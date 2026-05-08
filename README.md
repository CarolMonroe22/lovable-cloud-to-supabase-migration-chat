# lovable-cloud-supabase-migration-chat

A Claude skill for migrating a Lovable Cloud project to a standalone Supabase project — designed specifically for **claude.ai chat** (the consumer Claude product).

## What it does

Automates the data migration from Lovable Cloud to your own Supabase project:

- **Schema** — tables, enums, RLS policies, custom functions, triggers, sequences, indexes
- **Auth** — `auth.users` AND `auth.identities` with bcrypt hashes intact
- **Data** — every row in every table using a token-efficient `string_agg` SQL pattern
- **Storage** — all files via temporary edge function invoked through `pg_net`
- **Edge functions** — deployed with correct `verify_jwt` per function from `supabase/config.toml`

The frontend code migration is **delegated** to Claude Code (CLI or web) or your local terminal because claude.ai chat does not have GitHub access. See "Frontend migration" in `SKILL.md` for the workflow.

## Why a separate skill from the Code version

There's a sister skill, [`lovable-cloud-to-supabase-migration`](https://github.com/CarolMonroe22/lovable-cloud-to-supabase-migration), designed for Claude Code CLI and Claude Code on the web. That skill handles the full migration including the frontend code, because it has access to `gh`, `git`, and `rsync`.

This skill is **for users who are in claude.ai chat** and don't want to switch environments mid-task. It automates the hardest 95% of the migration in chat itself, then delegates the frontend code push to wherever the user is comfortable doing it.

| Scenario | Use this skill |
|---|---|
| You're in claude.ai chat | ✅ Yes |
| You're in Claude Code CLI | ❌ Use the original skill |
| You're in Claude Code on the web | ❌ Use the original skill |
| You don't have `gh` or `git` installed locally | ✅ Yes (delegate to Claude Code on the web for Phase 8) |

## What's better than the Code version

This skill incorporates **12 traps** discovered during real-world testing that the original Code skill missed. The most critical:

- **Trap 1**: Lovable changed its default tech stack to TanStack Start on May 6, 2026 — affecting all migrations of pre-May 6 projects. Detection logic via `package.json` is included.
- **Trap 8**: Custom functions like `handle_new_user` are not in the table schema and must be migrated separately or new user signups break.
- **Trap 9**: Sequences must be migrated with the correct `last_value` or duplicate orders/IDs occur.
- **Trap 11**: `auth.identities` must be migrated alongside `auth.users` or session recovery breaks.
- **Trap 5**: `verify_jwt` per edge function must be read from `supabase/config.toml` or webhooks (Stripe etc.) reject calls.

Full list with fixes in [`references/trampas.md`](./references/trampas.md).

## Installation

1. Download the latest release ZIP, or zip this folder yourself:
   ```bash
   zip -r lovable-cloud-supabase-migration-chat.zip lovable-cloud-supabase-migration-chat/
   ```
2. Open [claude.ai](https://claude.ai)
3. Click your profile → **Settings** → **Customize** → **Skills** (or **Capabilities** → **Skills** depending on plan)
4. Click **+** or **Upload a skill**
5. Upload the ZIP
6. Toggle the skill ON

## Required MCPs

Both must be connected in claude.ai chat (Settings → Connectors):

- **Lovable MCP** — `https://mcp.lovable.dev`
- **Supabase MCP** — `https://mcp.supabase.com/mcp`

The skill prompts you to connect them if they're missing.

## Usage

Open a new chat and describe what you want naturally:

> "I want to migrate my Lovable Cloud project to my own Supabase. The project is `<project-id>` in workspace `<workspace-id>`."

Or trigger it explicitly:

> "Use the `lovable-cloud-supabase-migration-chat` skill to migrate my Lovable project."

The skill walks through 9 phases, asking for ~14 approvals along the way (each `apply_migration` and write `execute_sql` requires user approval in claude.ai chat).

## Important notes

**Token consumption.** A typical migration consumes ~200K tokens — close to the full chat context window. The skill includes explicit compaction checkpoints after Phase 4 and Phase 6. For projects with 200+ storage files or 20+ edge functions, split into two sessions.

**This skill is in early testing.** Validated end-to-end against one project (POMPOM, with 13 tables, 175 storage files, 12 edge functions). The 12 traps are documented from that testing plus additional auditing of common Lovable patterns. **Test against a non-critical project first** before migrating production data.

## Limitations

- Frontend code migration requires Claude Code or your local terminal — see `SKILL.md` "Frontend migration" section for the workflow
- JWT secret preservation (so existing user sessions remain valid) requires manual dashboard copy — there's no API for it
- Edge function secrets must be set manually in the destination Supabase dashboard after deployment

## Contributing

If you find a 13th trap or hit a case the skill doesn't handle, please open an issue with:
- The Lovable project structure (counts of tables, edge functions, etc.)
- The exact failure point and error message
- Whether the failure is silent (data corruption) or loud (skill stops)

## License

MIT — see source files.

## Author

Carol Monroe ([@CarolMonroe22](https://github.com/CarolMonroe22)). Lovable Champion + Supabase SupaSquad member. Original Claude Code version of this skill: [`lovable-cloud-to-supabase-migration`](https://github.com/CarolMonroe22/lovable-cloud-to-supabase-migration).
