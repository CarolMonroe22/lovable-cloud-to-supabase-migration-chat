# Trampas — known traps and their fixes

This file documents the 12 traps discovered during real-world testing of Lovable Cloud → Supabase migrations. Each trap caused silent failures or broken migrations in the original `lovable-cloud-to-supabase-migration` skill (Claude Code version). They affect both that skill and this one.

Severity legend: 🔴 critical (breaks the app), 🟠 silent bug (data integrity issues), 🟡 degradation (performance or UX).

---

## Trap 1 — 🔴 Tech stack default changed (Phase 8)

**Discovered**: May 8, 2026. Lovable changed its default project template from Vite + React + React Router (`vite_react_shadcn_ts`) to TanStack Start (`tanstack_start_ts_2026-05-06`) on May 6, 2026.

**Impact**: When `Lovable MCP:create_project` is called without specifying `tech_stack`, the new empty project uses TanStack — but the source code being migrated is Vite-based. The `git push` succeeds, but Lovable cannot build it because the scaffold and the pushed code use different frameworks.

**Fix**: Detect the tech stack from the source `package.json` in Phase 1 and pass the correct value:

```python
# Pseudocode for the detection
package_json = read_file("package.json", source_project_id, latest_sha)
package_data = json.parse(package_json)

if package_data.name == "vite_react_shadcn_ts":
    tech_stack = "classic"  # Vite + React
elif "tanstack_start" in package_data.name:
    tech_stack = "modern"   # TanStack Start
else:
    tech_stack = "classic"  # Default fallback — most migrations are pre-May-2026 Vite projects
```

**Why "classic" is the safe default**: Any project being migrated FROM Lovable Cloud was created BEFORE the user decided to leave Lovable Cloud. That decision is made AFTER the project exists. So 99% of migrations are projects from before the May 6 default change.

---

## Trap 2 — 🟡 us-east-1 capacity outages (Phase 2)

**Discovered**: May 8, 2026 (during testing) and again April 25, 2026 (per Supabase status page). `us-east-1` has been disabled for project creation multiple times due to upstream provider capacity issues.

**Impact**: `Supabase:create_project` returns capacity errors when `region: "us-east-1"` is requested during an outage.

**Fix**: Default to `us-west-1` in Phase 2 unless the user explicitly requests another region for compliance/latency reasons. If `create_project` fails with capacity errors, automatically retry with `us-west-1` and inform the user.

```python
# Default and fallback
preferred_region = user_preference or "us-west-1"
fallback_region = "us-west-1"

try:
    create_project(region=preferred_region, ...)
except CapacityError:
    create_project(region=fallback_region, ...)
    notify_user(f"Region {preferred_region} had capacity issues — used {fallback_region} instead")
```

---

## Trap 3 — 🟡 pg_net not enabled by default (Phase 6)

**Discovered**: May 8, 2026. Newly created Supabase projects do NOT have `pg_net` enabled, even though Supabase advertises it as available.

**Impact**: Phase 6's storage migration relies on `net.http_post` to invoke the migrate-storage edge function. Without pg_net, the call fails with `function net.http_post does not exist`.

**Fix**: Enable pg_net at the start of Phase 3 (so it's ready for Phase 6):

```sql
CREATE EXTENSION IF NOT EXISTS pg_net WITH SCHEMA extensions;
```

Verify in Phase 9:
```sql
SELECT EXISTS(SELECT 1 FROM pg_extension WHERE extname='pg_net') AS pg_net_enabled;
```

**Note on rate limits**: pg_net handles up to 200 requests/second by default. For large storage migrations (>1000 files), batch into chunks of 25-50 files per http_post call.

---

## Trap 4 — 🟠 Private storage buckets not handled (Phase 6)

**Discovered**: While auditing the original skill. The skill assumes all buckets are public.

**Impact**: For private buckets, the URL pattern `https://<ref>.supabase.co/storage/v1/object/public/<bucket>/<path>` returns 400. The migrate-storage edge function gets a download error and skips the file. **Files appear migrated but are silently missing.**

**Fix**: In Phase 6, detect each bucket's `public` flag and use signed URLs for private buckets:

```sql
-- Detect bucket visibility
SELECT id, name, public FROM storage.buckets;
```

For public buckets:
```
url = 'https://<source_ref>.supabase.co/storage/v1/object/public/<bucket>/<path>'
```

For private buckets, generate signed URLs from the SOURCE Postgres:
```sql
-- This requires running on the SOURCE (Lovable Cloud) database
SELECT storage.create_signed_url('<bucket_id>', '<object_name>', 3600) AS signed_url;
```

The signed URL has a 1-hour TTL — make the migrate-storage call within that window.

---

## Trap 5 — 🟠 verify_jwt not read from config.toml (Phase 7)

**Discovered**: While reading POMPOM's `supabase/config.toml`. The original skill deploys all edge functions with the same `verify_jwt` value (or default), ignoring per-function configuration.

**Impact**: Webhooks (Stripe, custom integrations, public OAuth callbacks) require `verify_jwt: false`. If deployed with `verify_jwt: true`, they reject all incoming requests with 401. Conversely, user-facing functions deployed with `verify_jwt: false` become anonymously callable — a security risk.

**Fix**: In Phase 1, read `supabase/config.toml` from the source. Parse the `[functions.<name>]` sections to build a map:

```toml
# Example POMPOM config.toml
[functions.generate-concept]
verify_jwt = true

[functions.analyze-brand]
verify_jwt = false

[functions.create-demo-user]
verify_jwt = false
```

Build the mapping:
```
{
  "generate-concept": true,
  "analyze-brand": false,
  "create-demo-user": false,
  ...
}
```

In Phase 7, pass the correct value to each `Supabase:deploy_edge_function`:
```python
for fn_name, source_code in functions.items():
    deploy_edge_function(
        name=fn_name,
        files=[{"name": "index.ts", "content": source_code}],
        verify_jwt=config_toml_map.get(fn_name, True),  # default true if not specified
    )
```

---

## Trap 6 — 🟠 Edge function secrets not listed (Phase 7)

**Discovered**: While reading POMPOM's `extract-colors` edge function source. The original skill deploys edge functions but doesn't tell the user which environment variables (secrets) they need to set in the destination dashboard.

**Impact**: Functions deploy successfully but fail at runtime with "API key not configured" errors. User has to read every function's source to figure out the secrets — defeats the purpose of automation.

**Fix**: In Phase 7, grep each edge function for `Deno.env.get(...)` calls and aggregate the unique secret names. Exclude the auto-provided ones:

**Auto-provided by Supabase (DON'T list these):**
- `SUPABASE_URL`
- `SUPABASE_ANON_KEY`
- `SUPABASE_SERVICE_ROLE_KEY`
- `SUPABASE_DB_URL`

**Common ones the user needs to set manually:**
- `LOVABLE_API_KEY` — used by functions calling `ai.gateway.lovable.dev`
- `OPENAI_API_KEY` — direct OpenAI usage
- `ANTHROPIC_API_KEY` — direct Claude usage
- `STRIPE_SECRET_KEY` — payment processing
- `RESEND_API_KEY`, `SENDGRID_API_KEY` — email sending

Detection regex:
```regex
Deno\.env\.get\(["\047](\w+)["\047]\)
```

After Phase 7 completes, give the user a per-function breakdown:
```
extract-colors: requires LOVABLE_API_KEY
generate-concept: requires LOVABLE_API_KEY, OPENAI_API_KEY
generate-print-file: requires STRIPE_SECRET_KEY
...

Set these in: Supabase Dashboard → Settings → Edge Functions → Secrets
```

---

## Trap 7 — 🟠 URLs in JSONB and arrays not rewritten (Phase 5)

**Discovered**: While reviewing POMPOM data. URLs to the old Supabase storage hide in unexpected places — JSONB metadata fields, text arrays, configuration columns.

**Impact**: After migration, the database references storage at the OLD project ref. The frontend shows broken images even though the files were correctly migrated.

**Fix**: In Phase 5, scan ALL columns of types `text`, `text[]`, and `jsonb` in the public schema for the old ref pattern. Don't trust column names — URLs hide in unlikely places like `config`, `metadata`, `specs`.

**Discovery query**:
```sql
-- Find all text/jsonb columns that may contain URLs
SELECT 
  table_name, 
  column_name, 
  data_type
FROM information_schema.columns
WHERE table_schema = 'public'
  AND data_type IN ('text', 'ARRAY', 'jsonb', 'json');
```

For each candidate column, check for the old ref:
```sql
-- For text/text[] columns
SELECT count(*) FROM public.<table> WHERE <column>::text LIKE '%<old_ref>%';

-- For jsonb columns
SELECT count(*) FROM public.<table> WHERE <column>::text LIKE '%<old_ref>%';
```

Then rewrite:
```sql
-- text columns
UPDATE public.<table> SET <column> = REPLACE(<column>, '<old_ref>', '<new_ref>') WHERE <column> LIKE '%<old_ref>%';

-- jsonb columns (cast to text, replace, cast back)
UPDATE public.<table> 
SET <column> = REPLACE(<column>::text, '<old_ref>', '<new_ref>')::jsonb 
WHERE <column>::text LIKE '%<old_ref>%';
```

---

## Trap 8 — 🔴 Custom functions and triggers not migrated (Phase 3)

**Discovered**: While auditing POMPOM. The original skill copies tables, enums, RLS, but NOT the custom `CREATE FUNCTION` and `CREATE TRIGGER` definitions.

**Impact**: The most damaging case is `handle_new_user`, a trigger function on `auth.users INSERT` that auto-creates a `profiles` row. Without it, **new user signups succeed but the user has no profile**, and the app crashes when trying to load profile data.

Common functions seen in Lovable Cloud projects:
- `handle_new_user` — auto-create profile on signup (CRITICAL)
- `update_updated_at_column` — keep `updated_at` timestamp current
- Slug generators, order number generators, custom defaults
- Storage helpers (`storage.foldername`, etc.)

**Fix**: In Phase 1, scan for ALL custom functions:
```sql
SELECT 
  p.proname AS function_name, 
  pg_get_functiondef(p.oid) AS definition
FROM pg_proc p
JOIN pg_namespace n ON p.pronamespace = n.oid
WHERE n.nspname = 'public'
  AND p.proname NOT LIKE 'pg_%';
```

And ALL triggers:
```sql
SELECT 
  trigger_schema, trigger_name, event_object_table, 
  event_manipulation, action_timing, action_statement
FROM information_schema.triggers
WHERE trigger_schema IN ('public', 'auth')
  AND trigger_name NOT LIKE 'pg_%'
  AND trigger_name NOT LIKE 'RI_%';  -- exclude FK constraint triggers
```

In Phase 3, recreate them in this order:
1. Functions first (definitions don't depend on table state)
2. Then triggers (which reference the functions and tables already created)

---

## Trap 9 — 🔴 Sequences not migrated with correct `last_value` (Phase 3)

**Discovered**: While reviewing POMPOM's `order_number_seq`.

**Impact**: If a sequence is recreated fresh (defaulting to `last_value = 1`), but the source had reached `last_value = 50`, the migrated app will start generating order numbers like `POM-2026-0001` again — colliding with existing orders. Foreign key violations or duplicate keys ensue.

**Fix**: In Phase 1, capture both the sequence definition AND its current `last_value`:
```sql
SELECT 
  sequencename, 
  last_value, 
  start_value, 
  increment_by, 
  min_value, 
  max_value, 
  cycle
FROM pg_sequences 
WHERE schemaname = 'public';
```

In Phase 3 step 4, recreate with `setval`:
```sql
CREATE SEQUENCE public.order_number_seq
  START WITH <start_value>
  INCREMENT BY <increment_by>
  MINVALUE <min_value>
  MAXVALUE <max_value>
  <CYCLE | NO CYCLE>;

SELECT setval('public.order_number_seq', <last_value>, true);
```

The `true` argument means the next call to `nextval` will return `last_value + 1`, preserving the original sequence behavior.

---

## Trap 10 — 🟡 Custom indexes not recreated (Phase 3)

**Discovered**: While auditing POMPOM. Performance indexes (B-tree on commonly queried columns) are not recreated.

**Impact**: The app functions but slow queries become slower. Common patterns affected: `idx_brands_slug` for slug lookups, indexes on `created_at DESC` for activity feeds, indexes on FKs.

**Fix**: In Phase 1, scan for all non-PK, non-unique indexes:
```sql
SELECT schemaname, tablename, indexname, indexdef
FROM pg_indexes
WHERE schemaname = 'public'
  AND indexname NOT LIKE '%_pkey'
  AND indexname NOT LIKE '%_key';
```

In Phase 3 step 7, run each `indexdef` directly (it's a complete `CREATE INDEX` statement).

---

## Trap 11 — 🔴 auth.identities not migrated (Phase 4)

**Discovered**: Confirmed by Supabase official docs. The original skill migrates `auth.users` but skips `auth.identities`.

**Impact**: Login appears to work because the password hash is in `auth.users`. But:
- Password recovery flows fail (need identity for "forgot password")
- Session refresh tokens fail to validate
- OAuth users can't re-link accounts
- The `last_sign_in_at` field disappears

**Fix**: In Phase 4, migrate `auth.identities` AFTER `auth.users`. Per the Supabase troubleshooting docs:

> "If you migrate the data for `auth.users` and `auth.identities` you should have those users available to sign in with the same emails and passwords on a different project."

```sql
-- Read from source
SELECT id, user_id, provider, provider_id, identity_data, 
       created_at, updated_at, last_sign_in_at
FROM auth.identities;

-- Insert into destination (in same order as users were inserted)
INSERT INTO auth.identities (
  id, user_id, provider, provider_id, identity_data,
  created_at, updated_at, last_sign_in_at
) VALUES (...);
```

### Trap 11b — 🟡 JWT secret not preserved (Phase 4 optional)

**Impact**: Even with users + identities migrated, existing JWTs (active session tokens) become invalid because each Supabase project has a unique JWT secret. Users have to log in again.

**Fix** (manual, optional): If preserving sessions matters for the user, after Phase 4:
1. In source dashboard: Settings → API → JWT Secret → copy
2. In destination dashboard: Settings → API → JWT Secret → paste

This MUST be done via the dashboard — there's no API for it.

---

## Trap 12 — 🟠 Phase 9 verifies almost nothing (Phase 9)

**Discovered**: While reviewing the original skill's verify step.

**Impact**: The original Phase 9 only counts `tables` matched. It misses:
- Functions count mismatch (Trap 8)
- Triggers count mismatch (Trap 8)
- Sequences count mismatch (Trap 9)
- pg_net not enabled (Trap 3)
- Old ref still in data (Trap 7)
- auth.identities count (Trap 11)
- Storage objects count

**Fix**: Use the comprehensive audit query from `SKILL.md` Phase 9. Compare each row against the source values from Phase 1.

The audit should output a comparison table:
```
Category           Source  Destination  Status
─────────────────  ──────  ───────────  ────────
tables             N       N            OK
enums              N       N            OK
rls_policies       N       N            OK
functions_custom   N       N            OK or MISSING
triggers           N       N            OK or MISSING
sequences          N       N            OK or MISSING
auth_users         N       N            OK
auth_identities    N       N            OK or MISSING
storage_buckets    N       N            OK
storage_objects    N       N            OK or PARTIAL
edge_functions     N       N            OK or PARTIAL
pg_net_enabled     true    true         OK
old_ref_leaks      0       0            OK
```

If any row shows MISSING or PARTIAL, point the user to the specific phase that needs to be re-run.

---

## migrate-storage edge function template

This is the temporary edge function used in Phase 6. Deploy with `verify_jwt: false`.

```typescript
import "jsr:@supabase/functions-js/edge-runtime.d.ts";
import { createClient } from "jsr:@supabase/supabase-js@2";

const SUPABASE_URL = Deno.env.get("SUPABASE_URL")!;
const SUPABASE_SERVICE_ROLE_KEY = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;

interface FileSpec {
  source_url: string;
  bucket: string;
  path: string;
  content_type?: string;
}

interface RequestBody {
  files: FileSpec[];
}

Deno.serve(async (req) => {
  if (req.method !== "POST") {
    return new Response(JSON.stringify({ error: "POST only" }), {
      status: 405,
      headers: { "Content-Type": "application/json" },
    });
  }

  const body: RequestBody = await req.json();
  const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, {
    auth: { persistSession: false },
  });

  const results = [];
  for (const file of body.files) {
    try {
      const dl = await fetch(file.source_url);
      if (!dl.ok) {
        results.push({ path: file.path, ok: false, error: `Download failed: ${dl.status}` });
        continue;
      }
      const bytes = new Uint8Array(await dl.arrayBuffer());
      const contentType = file.content_type ?? dl.headers.get("content-type") ?? "application/octet-stream";

      const { error } = await supabase.storage
        .from(file.bucket)
        .upload(file.path, bytes, { contentType, upsert: true });

      if (error) {
        results.push({ path: file.path, ok: false, error: error.message });
      } else {
        results.push({ path: file.path, ok: true, bytes: bytes.byteLength });
      }
    } catch (e) {
      results.push({
        path: file.path,
        ok: false,
        error: e instanceof Error ? e.message : String(e),
      });
    }
  }

  const ok = results.filter((r) => r.ok).length;
  return new Response(JSON.stringify({ total: results.length, ok, failed: results.length - ok, results }), {
    headers: { "Content-Type": "application/json" },
  });
});
```

After Phase 6 completes successfully, **delete this function from the destination**:
```
Supabase MCP doesn't have a delete_edge_function tool, so the user must delete it via the dashboard:
Settings → Edge Functions → migrate-storage → Delete
```

---

## Phase 1 audit query (full version)

Use this in Phase 1 to capture everything that the original skill missed:

```sql
-- Run on SOURCE (Lovable Cloud) project via Lovable MCP query_database
WITH counts AS (
  SELECT 'tables' AS category, COUNT(*) AS n FROM information_schema.tables WHERE table_schema='public' AND table_type='BASE TABLE'
  UNION ALL SELECT 'enums', COUNT(*) FROM pg_type t JOIN pg_namespace n ON t.typnamespace=n.oid WHERE n.nspname='public' AND t.typtype='e'
  UNION ALL SELECT 'rls_policies', COUNT(*) FROM pg_policies WHERE schemaname='public'
  UNION ALL SELECT 'functions_custom', COUNT(*) FROM pg_proc p JOIN pg_namespace n ON p.pronamespace=n.oid WHERE n.nspname='public'
  UNION ALL SELECT 'triggers', COUNT(*) FROM information_schema.triggers WHERE trigger_schema IN ('public','auth') AND trigger_name NOT LIKE 'RI_%'
  UNION ALL SELECT 'sequences', COUNT(*) FROM pg_sequences WHERE schemaname='public'
  UNION ALL SELECT 'indexes_custom', COUNT(*) FROM pg_indexes WHERE schemaname='public' AND indexname NOT LIKE '%_pkey' AND indexname NOT LIKE '%_key'
  UNION ALL SELECT 'auth_users', COUNT(*) FROM auth.users
  UNION ALL SELECT 'auth_identities', COUNT(*) FROM auth.identities
  UNION ALL SELECT 'storage_buckets', COUNT(*) FROM storage.buckets
  UNION ALL SELECT 'storage_objects', COUNT(*) FROM storage.objects
)
SELECT * FROM counts ORDER BY category;
```

Save these counts. Use them as the comparison baseline in Phase 9.
