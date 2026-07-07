# Lovable Cloud to Supabase Migration (claude.ai chat)

[![Version](https://img.shields.io/badge/version-1.0.0-blue)](https://github.com/CarolMonroe22/lovable-cloud-to-supabase-migration-chat)
[![Tested](https://img.shields.io/badge/tested-2026--05--10-green)](https://github.com/CarolMonroe22/lovable-cloud-to-supabase-migration-chat)
[![License: MIT](https://img.shields.io/badge/license-MIT-yellow)](./LICENSE)

> **Update (July 2026):** Lovable Cloud now has official **Export**, **Pause** and **Remove** buttons, so the recommended flow changed. This chat skill still works, but check the [main migration skill v4](https://github.com/CarolMonroe22/lovable-cloud-to-supabase-migration) and the [current walkthrough](https://carolmonroe.com/blog/export-remove-lovable-cloud) first - the new same-project path is simpler and official.

Migrate your Lovable Cloud project to your own Supabase directly from claude.ai chat. No CLI, no terminal, no GitHub setup. Connect the MCPs, describe what you want, and the skill handles schema, data, auth users (with passwords and identities), storage, and edge functions.

| | |
|---|---|
| **From** | Lovable Cloud (managed database) |
| **To** | Your own Supabase project (full dashboard, custom emails, service role key) |
| **What moves** | Backend: schema, data, auth users with passwords and identities, storage, edge functions |
| **What doesn't** | Frontend code (delegated to Claude Code or your local terminal) |
| **How** | 9 phases with ~14 user approvals. The skill does the work, you click Allow |

## Install

```bash
npx skills add CarolMonroe22/lovable-cloud-to-supabase-migration-chat
```

You can also copy the repo URL and paste it directly into claude.ai or any agent that supports skills.

Or [download the .skill file](https://ajvkznmwjdeariuzlwoa.supabase.co/storage/v1/object/public/skills/lovable-cloud-supabase-migration-chat.skill) and upload it manually in claude.ai (Settings > Capabilities > Skills).

## Need the full migration including frontend?

If you want the frontend code migrated automatically too, use the [Claude Code version](https://github.com/CarolMonroe22/lovable-cloud-to-supabase-migration) of this skill. It handles everything end-to-end via `gh`, `git`, and `rsync`. Recommended if you have Claude Code installed.

## When to Use This Version

| Scenario | Use this skill? |
|---|---|
| You're in claude.ai chat | Yes |
| You only need to migrate the database | Yes |
| You don't have `gh` or `git` installed | Yes (delegate frontend to Claude Code on the web) |
| You're in Claude Code CLI or web | No, use the [Code version](https://github.com/CarolMonroe22/lovable-cloud-to-supabase-migration) |

## Required MCPs

Both must be connected in claude.ai chat (Settings > Connectors):

| MCP | URL | Setup |
|---|---|---|
| Lovable MCP | `https://mcp.lovable.dev` | Add as custom connector |
| Supabase MCP | `https://mcp.supabase.com/mcp` | Already in the directory, click Configure |

If you haven't set them up yet, follow the [MCP setup guide](https://carolmonroe.com/blog/connect-lovable-supabase-mcp-to-claude).

## How It Works

The skill runs 9 phases in order. Each phase asks for user approval before making changes.

| Phase | What happens |
|---|---|
| 1. Scan Source | Reads schema, data, auth users, identities, functions, triggers, sequences, indexes, storage, config.toml, and tech stack |
| 2. Create Destination | Creates a new Supabase project (defaults to us-west-1) |
| 3. Apply Schema | Recreates tables, enums, constraints, sequences with last_value, custom functions, triggers, indexes, and RLS policies |
| 4. Auth Users | Imports users with original passwords AND identities |
| 5. Insert Data | Moves all data using token-efficient string_agg pattern, rewrites URLs in all text and jsonb columns |
| 6. Storage | Migrates files from public and private buckets via temporary edge function + pg_net |
| 7. Edge Functions | Deploys functions with correct verify_jwt per function and reports required secrets |
| 8. Frontend Code | Delegated - generates instructions for Claude Code on the web or local terminal |
| 9. Verify | Compares 12 categories between source and destination, scans for old URL leaks |

## What Gets Migrated

| Component | Status |
|---|---|
| Database schema (tables, enums, relationships, constraints) | Fully automated |
| Custom functions (handle_new_user, etc.) | Fully automated |
| Custom triggers | Fully automated |
| Sequences with current last_value | Fully automated |
| Custom indexes | Fully automated |
| Row-level security policies | Fully automated |
| All table data | Fully automated |
| URL rewriting (text, jsonb, arrays) | Fully automated |
| Auth users with original passwords | Fully automated |
| Auth identities | Fully automated |
| Storage buckets (public and private) | Fully automated |
| Edge functions with correct verify_jwt | Fully automated |
| Edge function secrets inventory | Listed for manual setup |
| Frontend code | Delegated (see Phase 8) |

## 12 Documented Traps

This skill incorporates 12 traps discovered during real-world testing. Each one is an explicit step in the migration flow.

| Trap | What happens if missed |
|---|---|
| TanStack detection | Destination project uses wrong framework, build fails |
| auth.identities | Password recovery and session refresh break |
| Custom functions | handle_new_user missing, new signups create no profile |
| Sequences | Duplicate IDs and order number collisions |
| Private buckets | Files silently missing |
| verify_jwt | Webhooks return 401 or functions become public |
| Edge function secrets | Functions deploy but fail at runtime |
| URL rewriting | Broken images despite files being migrated |
| Custom indexes | Slow queries |
| pg_net | Storage migration fails |
| us-east-1 outages | Project creation fails |
| Phase 9 verification | Migration appears complete but pieces are missing |

Full details with fixes in [`references/trampas.md`](./references/trampas.md).

## Good to Know

- **Token consumption.** A typical migration consumes ~200K tokens. The skill includes compaction checkpoints after Phase 4 and Phase 6. For projects with 200+ storage files or 20+ edge functions, split into two sessions.
- **Your users keep their passwords.** The skill copies the original bcrypt hashes and identities, so everyone logs in with the same credentials.
- **Classic to classic, modern to modern.** The skill detects whether your project uses Vite or TanStack Start and creates the destination with the correct stack. It does not convert between stacks.
- **This skill is in early testing.** Validated end-to-end against one project (POMPOM, with 13 tables, 175 storage files, 12 edge functions). Test against a non-critical project first.

## Limitations

- Frontend code migration requires Claude Code or your local terminal
- JWT secret preservation (so existing user sessions remain valid) requires manual dashboard copy
- Edge function secrets must be set manually in the destination Supabase dashboard after deployment

## Resources

| Resource | Link |
|---|---|
| MCP setup guide for claude.ai chat | [carolmonroe.com/blog/connect-lovable-supabase-mcp-to-claude](https://carolmonroe.com/blog/connect-lovable-supabase-mcp-to-claude) |
| Migration guide (blog post) | [carolmonroe.com/blog/lovable-cloud-mcp-migration](https://carolmonroe.com/blog/lovable-cloud-mcp-migration) |
| Claude Code version of this skill | [github.com/CarolMonroe22/lovable-cloud-to-supabase-migration](https://github.com/CarolMonroe22/lovable-cloud-to-supabase-migration) |
| Disconnect guide | [carolmonroe.com/blog/disconnect-lovable-cloud](https://carolmonroe.com/blog/disconnect-lovable-cloud) |
| Lovable MCP docs | [docs.lovable.dev/integrations/mcp-servers](https://docs.lovable.dev/integrations/mcp-servers) |
| Supabase MCP docs | [supabase.com/docs/guides/getting-started/mcp](https://supabase.com/docs/guides/getting-started/mcp) |

## Contributing

If you find a 13th trap or hit a case the skill doesn't handle, please open an issue with:
- The Lovable project structure (counts of tables, edge functions, etc.)
- The exact failure point and error message
- Whether the failure is silent (data corruption) or loud (skill stops)

## License

MIT

## Author

[Carol Monroe](https://carolmonroe.com) - Lovable Champion and Supabase SupaSquad Member

- GitHub: [@CarolMonroe22](https://github.com/CarolMonroe22)
- X: [@carolmonroe](https://x.com/carolmonroe)
- LinkedIn: [carolmonroe](https://linkedin.com/in/carolmonroe)
