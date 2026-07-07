---
name: lovable-cloud-supabase-migration-chat
description: Migrate a Lovable Cloud project to a standalone Supabase project from claude.ai chat (the consumer Claude product, not Claude Code). Automates schema, data, auth, storage, and edge functions migration via Lovable MCP and Supabase MCP. Since July 2026 the endgame is official - after verification, the user removes Lovable Cloud and connects their own Supabase to the SAME Lovable project, so no code push is needed in the default flow. Use this skill when the user asks to migrate, export, or move a Lovable Cloud project to their own Supabase, especially when they mention being in claude.ai chat or not having Claude Code installed. Keywords - migrate Lovable Cloud, export from Lovable, leave Lovable Cloud, Lovable to Supabase, migrate Lovable database.
license: MIT
---

# Lovable Cloud → Supabase migration (claude.ai chat)

This skill migrates a Lovable Cloud project to a standalone Supabase project from **claude.ai chat** (the consumer Claude product). It automates the hardest 95% of the work: schema, data, auth, storage, and edge functions. The last step is official since July 2026: once verification passes, the user removes Lovable Cloud and connects the new Supabase to the **same Lovable project** from the dashboard — no code push, no new project. The old new-project-plus-code-push route survives only as a fallback for users who must keep the Cloud project untouched.

## When to use

Trigger this skill when the user is in **claude.ai chat** and asks to migrate a Lovable Cloud project to their own Supabase. Common phrasings: "migrate my Lovable project to Supabase", "export from Lovable Cloud", "leave Lovable Cloud", "move my data out of Lovable".

If the user is in **Claude Code CLI**, **Claude Code on the web**, or **Claude Desktop**, point them to the original `lovable-cloud-to-supabase-migration` skill instead, which handles the frontend code automatically via `gh` and `git`.

## Required MCPs

This skill requires both MCPs to be connected in claude.ai chat (Settings → Connectors):

- **Lovable MCP** (`https://mcp.lovable.dev`) — for reading the source project
- **Supabase MCP** (`https://mcp.supabase.com/mcp`) — for creating and writing the destination

If either is missing, stop and ask the user to connect it before proceeding.

## What this skill automates

| Phase | What happens | Automation level |
|---|---|---|
| 1. Scan source | Audit the Lovable Cloud project structure | Fully automatic |
| 2. Create destination | Provision new Supabase project | Asks user for ~2 approvals |
| 3. Schema | Recreate tables, enums, RLS, **functions, triggers, sequences, indexes** | Asks for ~3 approvals |
| 4. Auth users | Migrate `auth.users` AND `auth.identities` with bcrypt hashes intact | Asks for ~2 approvals |
| 5. Insert data | Copy all rows table by table using `string_agg` SQL pattern | Asks for ~N approvals (1 per table) |
| 6. Storage | Deploy temporary edge function + invoke via `pg_net` | Asks for ~3 approvals |
| 7. Edge functions | Read source code, deploy to destination with correct `verify_jwt` | Asks for ~M approvals (1 per function) |
| 8. Verify | Run a comprehensive audit query BEFORE anything is removed | Fully automatic |
| 9. The switch | Remove Lovable Cloud + connect own Supabase on the SAME project | Manual user step (dashboard) |

## Important: token consumption and compaction

Migrating a typical project consumes approximately **200K tokens** — close to the full context window of claude.ai chat.

**Recommended compaction checkpoints:**
- After Phase 4 (Auth) — discharge source scan details
- After Phase 6 (Storage) — before loading edge function code

Use the `/compact` command in the chat menu when prompted, or the model will auto-compact at ~95% capacity.

**For projects larger than 200 storage files OR 20+ edge functions**, split the migration into two sessions:
- Session 1: Phases 1–6 (schema + data + storage)
- Session 2: Phases 7–8 (edge functions + verification), then the switch

## The endgame: same project (default) or new project (fallback)

**Default since July 2026**: the frontend never moves. Lovable added official **Export**, **Pause** and **Remove** buttons to Cloud. After this skill migrates the backend and verification passes, the user removes Lovable Cloud from the source project and connects their own Supabase to the **same Lovable project** via the dashboard. Same code, same repo, same URL — only the backend changes. Phase 9 walks through it.

**Sacred order — never violate it**: migrate → verify (Phase 8, all green) → download the official Export backup → Remove Cloud → Connect. Removing Cloud deletes its database AND storage permanently, including any un-downloaded exports. Do not let the user remove anything while verification has a single red row.

**Fallback: new project + code push.** Only needed when the user must keep the original Cloud project untouched (e.g., staging stays live while they test the copy). The frontend codebase (typically 200+ files) cannot be faithfully reproduced from claude.ai chat:
- No GitHub MCP exists for the consumer Claude product
- `Lovable MCP:remix_project` inherits its own fresh Cloud state — two Cloud instances mid-migration, unclear where data lives. Do not use remix for migration
- `Lovable MCP:create_project` with an `initial_message` makes the agent rewrite from scratch (not faithful to the original)
- Reading and rewriting 200+ files individually is prohibitively token-expensive

For the fallback, delegate the code push:

**Option 1: Claude Code on the web (fastest)**
Direct the user to [claude.ai/code](https://claude.ai/code), where they can:
1. Connect their GitHub account (one-time setup)
2. Run the original `lovable-cloud-to-supabase-migration` skill (fresh-project path)
3. The skill handles `gh repo clone`, `rsync`, `git push` automatically in the cloud VM

**Option 2: Local terminal**
If the user has `gh` and `git` installed, generate the exact commands for them to run locally. See `references/trampas.md` for the command template.

In both fallback options, after the code lands in the new Lovable repo, send a follow-up `Lovable MCP:send_message` to trigger a rebuild against the new Supabase.

## Critical traps

This skill incorporates 12 traps discovered through end-to-end testing on real projects. **Read `references/trampas.md` before starting Phase 1** — these traps cause silent failures that the original skill (Code version) didn't catch.

The most critical:
- **Trap 1**: Lovable's default tech_stack changed to TanStack Start on May 6, 2026. Use `tech_stack: "classic"` for projects created before that date (most migrations).
- **Trap 8**: Custom functions like `handle_new_user` are not in the table schema and must be migrated separately or new user signups break.
- **Trap 11**: `auth.identities` must be migrated alongside `auth.users` or session recovery breaks.
- **Trap 5**: `verify_jwt` per edge function must be read from `supabase/config.toml` or webhooks (Stripe etc.) reject calls.

## Migration workflow

The migration follows phases 1–9 in order. Each phase below describes the expected tool calls and SQL. Detailed templates and trap fixes are in `references/trampas.md`.

### Phase 1: Scan source (~6K tokens)

Inputs needed from user:
- Lovable project ID (or workspace and project name to look up)
- Lovable workspace ID

Tool calls:
1. `Lovable MCP:list_workspaces` — confirm workspace
2. `Lovable MCP:get_project` — get latest commit SHA and project metadata
3. `Lovable MCP:read_file` for `package.json` at the latest SHA — **detect tech stack**:
   - If `name === "vite_react_shadcn_ts"` → use `tech_stack: "classic"` if the Phase 9 fallback is taken
   - If `name` contains `tanstack_start` → `tech_stack: "modern"`
   - Default fallback: `"classic"` (most migrations are pre-May-2026 projects)
4. `Lovable MCP:read_file` for `supabase/config.toml` — **read `verify_jwt` per function** for Phase 7
5. `Lovable MCP:query_database` — run the audit queries:
   ```sql
   -- Tables, enums, RLS, functions, triggers, sequences, indexes, auth, storage
   -- Full query in references/trampas.md, section "Phase 1 audit query"
   ```

Output a structured summary to the user before proceeding:
```
Source project: <name>
Tech stack: classic | modern
Tables: N | Enums: N | RLS policies: N
Custom functions: N (handle_new_user: yes/no)
Custom triggers: N
Custom sequences: N
auth.users: N | auth.identities: N
Storage buckets: N (public: N, private: N)
Storage files: N (total: X MB)
Edge functions: N (config.toml verify_jwt mapping: ...)
```

### Phase 2: Create destination Supabase project (~2K tokens)

1. `Supabase:list_organizations` — confirm org
2. `Supabase:get_cost` — show user the cost of new project ($10/mo for Free → Pro upgrade)
3. **Tell the user**: "This will create a new Supabase project at $10/mo. Confirm to proceed."
4. `Supabase:confirm_cost` — get confirmation token
5. `Supabase:create_project` with parameters:
   - `name`: derive from user's preference
   - `organization_id`: from step 1
   - `region`: **default to `us-west-1`** (us-east-1 has had multiple capacity outages — see Trap 2)
   - `confirm_cost_id`: from step 4
6. Wait for project to be `ACTIVE_HEALTHY`. Save the new project ref.

If `us-east-1` was specifically requested and create_project fails with capacity errors, retry with `us-west-1` and inform the user.

### Phase 3: Migrate schema (~10K tokens)

This phase replicates the source schema into the destination. **Critical**: do not just copy `CREATE TABLE` — replicate functions, triggers, sequences, and indexes too.

1. **Enable pg_net first** (Trap 3):
   ```sql
   CREATE EXTENSION IF NOT EXISTS pg_net WITH SCHEMA extensions;
   ```
2. `Supabase:apply_migration` — create all enums:
   ```sql
   CREATE TYPE public.<enum_name> AS ENUM (...);
   ```
3. `Supabase:apply_migration` — create all tables with their columns, FKs, and constraints (read schema from source via `query_database` introspection on `information_schema.columns`)
4. `Supabase:apply_migration` — create custom **sequences** with the correct `last_value`:
   ```sql
   CREATE SEQUENCE public.order_number_seq;
   SELECT setval('public.order_number_seq', <last_value_from_source>);
   ```
5. `Supabase:apply_migration` — create custom **functions** (use `pg_get_functiondef` from source). This includes critical functions like `handle_new_user`.
6. `Supabase:apply_migration` — create **triggers** that wire functions to tables.
7. `Supabase:apply_migration` — create **custom indexes** (Trap 10):
   ```sql
   CREATE INDEX idx_brands_slug ON public.brands USING btree (slug);
   ```
8. `Supabase:apply_migration` — create RLS policies:
   ```sql
   ALTER TABLE public.<table> ENABLE ROW LEVEL SECURITY;
   CREATE POLICY "<name>" ON public.<table> FOR <action> USING (...);
   ```

After Phase 3, the destination has the complete schema structure. Verify with:
```sql
SELECT 
  (SELECT count(*) FROM pg_proc p JOIN pg_namespace n ON p.pronamespace=n.oid WHERE n.nspname='public') AS functions,
  (SELECT count(*) FROM information_schema.triggers WHERE trigger_schema IN ('public','auth')) AS triggers;
```


### Phase 4: Migrate auth users (~3K tokens)

**Critical**: migrate BOTH `auth.users` AND `auth.identities` (Trap 11). Without identities, login partially works but session recovery breaks.

1. `Lovable MCP:query_database` — read both tables:
   ```sql
   -- Get users
   SELECT id, email, encrypted_password, email_confirmed_at, raw_user_meta_data, ...
   FROM auth.users;
   ```
   Then separately:
   ```sql
   SELECT id, user_id, provider, provider_id, identity_data, created_at, last_sign_in_at
   FROM auth.identities;
   ```

2. `Supabase:apply_migration` — insert users with the bcrypt hashes intact (`$2a$10$...` format, 60 chars):
   ```sql
   INSERT INTO auth.users (
     instance_id, id, aud, role, email, encrypted_password,
     email_confirmed_at, raw_user_meta_data, raw_app_meta_data,
     created_at, updated_at, ...
   ) VALUES
     ('00000000-0000-0000-0000-000000000000', '<uuid>', 'authenticated', 'authenticated', '<email>', '<hash>', ...);
   ```

3. `Supabase:apply_migration` — insert identities:
   ```sql
   INSERT INTO auth.identities (
     id, user_id, provider, provider_id, identity_data, created_at, last_sign_in_at
   ) VALUES (...);
   ```

4. **Optional: preserve JWT secret** (Trap 11b). If the user wants existing sessions to remain valid (no forced re-login), they need to manually copy the JWT secret from source to destination via the dashboard. Mention this only if relevant to their use case.

### Phase 5: Migrate row data (~65K tokens — the heaviest phase)

**Key technique** (sole web advantage over Code): use PostgreSQL's `string_agg` + `quote_literal` to make Postgres generate its own INSERT statements. This eliminates the need for Python intermediates and is more token-efficient.

For each table (in dependency order — tables with no FKs first, then dependent ones):

1. `Lovable MCP:query_database` on the source — generate the INSERT SQL:
   ```sql
   SELECT string_agg(
     'INSERT INTO public.<table> (<col1>, <col2>, ...) VALUES (' ||
     quote_literal(<col1>::text) || '::<type>, ' ||
     COALESCE(quote_literal(<col2>::text), 'NULL') || '::<type>, ' ||
     '...' ||
     ');',
     E'\n'
   ) AS insert_sql
   FROM public.<table>;
   ```

   **Type casts matter**: include `::uuid`, `::jsonb`, `::timestamptz`, etc. for non-text columns. NULLs need special handling with COALESCE.

2. Take the `insert_sql` result and pass it directly to `Supabase:apply_migration` on the destination.

3. **Batching for large tables**: if a table has more than ~500 rows, the resulting SQL may exceed message size limits. Split with `LIMIT/OFFSET`:
   ```sql
   ... FROM public.<table> ORDER BY <pk> LIMIT 500 OFFSET 0;
   ```

4. **URL rewriting** (Trap 7): if any column type `text`, `text[]`, or `jsonb` contains references to the old Supabase ref (e.g., `https://<old_ref>.supabase.co/storage/...`), rewrite them in the destination:
   ```sql
   UPDATE public.<table>
   SET <column> = REPLACE(<column>, '<old_ref>', '<new_ref>')
   WHERE <column> LIKE '%<old_ref>%';
   ```

   **Don't only check `*_url` columns** — check all text/jsonb columns. URLs hide in JSONB metadata fields.

After Phase 5, **suggest compaction** to the user before proceeding to Phase 6:
> "We've migrated the schema, auth, and data. Context is at ~85K tokens. Run `/compact` in the chat menu to reduce context before storage migration, then continue with Phase 6."

### Phase 6: Migrate storage (~27K tokens)

**Key technique** (web is BETTER than Code here): use `pg_net` to invoke an edge function from Postgres itself. This is cleaner than Code's curl approach.

1. `Lovable MCP:query_database` — list buckets and detect public/private flag (Trap 4):
   ```sql
   SELECT id, name, public FROM storage.buckets;
   ```

2. `Supabase:apply_migration` — create buckets in the destination with their RLS policies:
   ```sql
   INSERT INTO storage.buckets (id, name, public)
   VALUES ('<bucket_id>', '<name>', <public_bool>);
   
   -- Recreate the matching storage.objects RLS policies from source
   ```

3. `Supabase:deploy_edge_function` — deploy the temporary `migrate-storage` function. Full code template in `references/trampas.md` ("migrate-storage edge function template"). Pass `verify_jwt: false`.

4. Generate the migration payload via SQL:
   ```sql
   SELECT jsonb_build_object(
     'files', jsonb_agg(
       jsonb_build_object(
         'source_url', 'https://<source_ref>.supabase.co/storage/v1/object/public/' || bucket_id || '/' || name,
         'bucket', bucket_id,
         'path', name,
         'content_type', metadata->>'mimetype'
       )
     )
   ) AS payload
   FROM storage.objects
   WHERE bucket_id = '<bucket_id>';
   ```

   **For private buckets** (Trap 4): use signed URLs instead:
   ```sql
   -- Generate signed URL for each object via storage.create_signed_url(bucket_id, name, 3600)
   ```

5. `Supabase:execute_sql` on destination — invoke the edge function via pg_net:
   ```sql
   SELECT net.http_post(
     url := 'https://<dest_ref>.supabase.co/functions/v1/migrate-storage',
     headers := '{"Content-Type": "application/json"}'::jsonb,
     body := '<payload from step 4>'::jsonb
   ) AS request_id;
   ```

6. Wait 5-30 seconds depending on file count and check the response:
   ```sql
   SELECT id, status_code, content::jsonb, error_msg, created
   FROM net._http_response
   WHERE id = <request_id>;
   ```

7. **For large buckets** (>50 files): split into chunks of 25 files each to stay under edge function timeout limits. Loop the pg_net calls with different request IDs.

8. After migration completes, **delete the migrate-storage edge function** from the destination (it's a temporary helper).

After Phase 6, **suggest compaction again** before Phase 7:
> "Storage migrated. Context is now ~113K tokens. Run `/compact` before Phase 7 — we're about to load 12+ edge functions."


### Phase 7: Migrate edge functions (~90K tokens — the second-heaviest)

For each edge function in the source `supabase/functions/<name>/index.ts`:

1. `Lovable MCP:read_file` at path `supabase/functions/<name>/index.ts`

2. **Grep for secrets** the function uses (Trap 6):
   ```
   grep -E 'Deno\.env\.get\(["\047](\w+)["\047]\)' index.ts
   ```
   Track which secrets are needed (excluding the Supabase auto-provided ones: `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`).

3. `Supabase:deploy_edge_function`:
   - `name`: same as source
   - `files`: `[{ name: "index.ts", content: <read_file output> }]`
   - **`verify_jwt`**: read from `supabase/config.toml` (Trap 5) — typically `true` for user-facing functions, `false` for webhooks.

4. Repeat for each function. **For projects with 20+ functions**, this phase alone can use 100K+ tokens — strongly consider splitting into a separate session.

After all functions are deployed, give the user the **list of secrets they need to set manually** in the destination Supabase dashboard:
> "Deployed N edge functions. You need to set these secrets in your Supabase dashboard (Settings → Edge Functions → Secrets):
> - LOVABLE_API_KEY (used by extract-colors, generate-concept)
> - OPENAI_API_KEY (used by ...)
> - STRIPE_SECRET_KEY (used by ...)
> 
> The functions will fail until these are set."

### Phase 8: Verify (~5K tokens)

Run this BEFORE anything is removed. The source Cloud project must still be alive and untouched when this phase runs — if verification fails, nothing has been lost.

Run a comprehensive audit query on the destination and compare with the source scan from Phase 1. This catches the things the original Code skill's verification missed (Trap 12).

```sql
-- Comprehensive audit query — full version in references/trampas.md
SELECT 'tables' AS category, COUNT(*) AS count FROM information_schema.tables WHERE table_schema='public'
UNION ALL SELECT 'enums', COUNT(*) FROM pg_type t JOIN pg_namespace n ON t.typnamespace=n.oid WHERE n.nspname='public' AND t.typtype='e'
UNION ALL SELECT 'rls_policies', COUNT(*) FROM pg_policies WHERE schemaname='public'
UNION ALL SELECT 'functions_custom', COUNT(*) FROM pg_proc p JOIN pg_namespace n ON p.pronamespace=n.oid WHERE n.nspname='public' AND p.proname NOT LIKE 'pg_%'
UNION ALL SELECT 'triggers', COUNT(*) FROM information_schema.triggers WHERE trigger_schema IN ('public','auth')
UNION ALL SELECT 'sequences', COUNT(*) FROM pg_sequences WHERE schemaname='public'
UNION ALL SELECT 'auth_users', COUNT(*) FROM auth.users
UNION ALL SELECT 'auth_identities', COUNT(*) FROM auth.identities
UNION ALL SELECT 'storage_buckets', COUNT(*) FROM storage.buckets
UNION ALL SELECT 'storage_objects', COUNT(*) FROM storage.objects
UNION ALL SELECT 'pg_net_enabled', CASE WHEN EXISTS(SELECT 1 FROM pg_extension WHERE extname='pg_net') THEN 1 ELSE 0 END;
```

Then **scan for old Supabase ref leaks** — any URL still pointing at the source ref means the data wasn't fully rewritten:
```sql
-- For each text/jsonb column in public schema, check for old ref
-- Full discovery query in references/trampas.md
SELECT count(*) FROM <table> WHERE <col> LIKE '%<old_ref>%';
```

Present the audit results to the user as a comparison table:
```
Category           Source  Destination  Status
─────────────────  ──────  ───────────  ────────
tables             N       N            OK
enums              N       N            OK
functions_custom   N       N            OK   <-- if MISSING, Phase 3 step 5 was skipped
auth_identities    N       N            OK   <-- if MISSING, Phase 4 step 3 was skipped
...
```

If any row shows MISSING or PARTIAL, point the user to the specific phase that needs to be re-run.

**GATE DECISION**: all green → proceed to Phase 9. Any red → STOP. Fix and re-verify. Cloud is still alive and untouched; nothing has been lost. Do NOT let the user remove Cloud with a red gate. There is no rush — some users sit at the green gate for days. That's fine.

Also add functional checks beyond the counts:
- Log in with a REAL migrated user and their ORIGINAL password
- Open a real storage file in the browser
- Invoke one edge function

### Phase 9: The switch — Remove Cloud + Connect (dashboard, HUMAN steps)

These are dashboard clicks the user does themselves. Walk them through, one step at a time:

1. **Download the official Export backup first.** Lovable dashboard → Cloud tab → Export project data. Download the `.backup` file locally. This is the insurance policy — the export is saved INTO the Cloud storage that's about to be deleted, so an un-downloaded export dies with Cloud.

2. **Remove Lovable Cloud.** Cloud tab > Overview > Advanced settings > Remove Lovable Cloud. Two checkboxes + type the project name. This permanently deletes the Cloud instance INCLUDING its storage.

3. **Connect the own Supabase.** The Cloud tab transforms. At the bottom: "Already have a Supabase project? Connect it here." > authorize the Supabase org > pick the new project > Connect. Lovable auto-rewrites `.env` with the new URL + key.

4. **Fix the integration overwrite.** The connect step may refresh `src/integrations/supabase/client.ts` (or equivalent) with a template — an error page right after connecting is THIS, not data loss. Send the Lovable agent (via `Lovable MCP:send_message`, in English):

   > Review my Supabase connection: confirm the app points to my new Supabase project, check that auth, database queries, storage and edge functions all work against it, and list anything still referencing the old backend.
   >
   > Important context: the database, users, storage files and edge functions ALREADY EXIST in the new project, they were migrated. Do NOT create tables, run migrations, re-seed data, or rewrite files beyond the connection wiring. Fix wiring only.

   The Do NOTs matter: the agent is eager and a bare "fix my connection" can trigger re-migrations or file rewrites.

### Phase 9-fallback: new project + code push (only if Cloud must stay untouched)

Generate this guidance for the user:

> **Frontend migration (manual)**
> 
> Your destination Supabase project is ready: `<dest_ref>`
> Source Lovable project: `<source_id>` at commit `<latest_sha>`
> 
> **Recommended: Claude Code on the web**
> 1. Open https://claude.ai/code
> 2. Connect your GitHub account (Settings → GitHub App)
> 3. Run the `lovable-cloud-to-supabase-migration` skill there (fresh-project path)
> 4. Provide it the same source and destination IDs above
> 5. The Claude Code skill handles `gh repo clone`, `rsync`, and `git push` automatically
> 
> **Alternative: your local terminal**
> If you have `gh` and `git` installed, run these commands:
> ```bash
> # 1. Clone the original Lovable repo
> gh repo clone <github_username>/<lovable_repo_name> ~/lovable-old
> 
> # 2. Create new empty Lovable project (we'll do this via MCP for you)
> # New project ID: <new_lovable_id>
> # New repo: <github_username>/<new_lovable_repo>
> 
> # 3. Clone the new (empty) repo
> gh repo clone <github_username>/<new_lovable_repo> ~/lovable-new
> 
> # 4. Copy code over (excluding Lovable Cloud-specific files)
> rsync -av --exclude='.git' --exclude='.lovable' --exclude='supabase/.temp' \
>   ~/lovable-old/ ~/lovable-new/
> 
> # 5. Update .env or src/integrations/supabase/client.ts to point to <dest_ref>
> 
> # 6. Commit and push
> cd ~/lovable-new && git add -A && git commit -m "Migrate from Lovable Cloud to Supabase" && git push
> ```

**This skill CAN automate two fallback steps** (creating the new empty Lovable project and triggering rebuild). Offer them to the user:

1. `Lovable MCP:create_project`:
   - `description`: "Migration target for <source_name>"
   - `workspace_id`: same workspace
   - **`tech_stack`**: from Phase 1 detection — usually `"classic"` (Trap 1)
   - `visibility`: `private`
   - **No `initial_message`** — we want it as empty as possible, the user will push code via git

2. After user reports they pushed the code, `Lovable MCP:send_message`:
   - `project_id`: new project ID
   - `message`: "The codebase has been updated via GitHub. Please rebuild against the connected Supabase database."

## Closing the migration

When verification passed and the switch is done:
1. Tell the user the same Lovable project now runs on their own Supabase
2. Remind them to set edge function secrets in the dashboard (from Phase 7's list) — functions fail until these are set
3. Recreate OAuth providers if used (Google etc.): client ID/secret + redirect URI `https://<new_ref>.supabase.co/auth/v1/callback` + Site URL
4. Offer to delete the temporary `migrate-storage` edge function from Phase 6
5. Tell them to delete the local `.backup` once everything is verified (it holds password hashes)

## Errors and recovery

- **`apply_migration` rejected**: most often a missing dependency (e.g., trying to create a trigger before its function). Re-order following Phase 3's exact sequence: enums → tables → sequences → functions → triggers → indexes → RLS.
- **`pg_net` returns null content**: the edge function may have crashed. Check `error_msg` in `net._http_response`. Common cause: source URL is private but you used public URL format.
- **`create_project` fails with capacity error in us-east-1**: retry with `us-west-1` (Trap 2).
- **Edge function 401 Unauthorized**: `verify_jwt` doesn't match the calling pattern. Webhooks must have `verify_jwt: false` (Trap 5).
- **New users sign up but no profile is created**: the `handle_new_user` trigger wasn't migrated (Trap 8). Re-run Phase 3 step 5 + step 6.
- **Error page right after connecting the own Supabase**: the connect step refreshed the integration client file with a template (Phase 9 step 4). This is wiring, not data loss — send the Lovable agent the fix-wiring-only message from Phase 9.
