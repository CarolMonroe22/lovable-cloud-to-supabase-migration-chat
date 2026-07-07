# Changelog

All notable changes to this skill, in human language.

## v2.0.0 - 2026-07-07 - "The endgame is official now"

In July 2026 Lovable shipped official **Export**, **Pause** and **Remove** buttons
for Cloud, and that changed how this skill ends. The backend migration (phases 1-7)
is untouched - MCP is still the data path from claude.ai chat, since the official
`.backup` export can't be restored without a local `pg_restore`.

### Changed
- **New endgame: the same-project switch.** After verification, the user removes
  Lovable Cloud and connects their own Supabase to the SAME Lovable project.
  No new project, no code push, same URL. The frontend never moves.
- **Verify now runs BEFORE anything is removed.** Phases 8 and 9 swapped:
  Phase 8 is the full audit + functional checks + gate decision, Phase 9 is the
  switch. Sacred order: migrate, verify all green, download the official Export
  backup, Remove, Connect. Never remove Cloud with a red gate.
- **The old new-project + code-push route is now the fallback**, kept for users
  who must leave the original Cloud project untouched.
- The closing section no longer claims Lovable Cloud "cannot be disabled" -
  Remove is an official button now (Cloud tab > Overview > Advanced settings).
- Remix warning reworded: remix inherits its own fresh Cloud state (two Cloud
  instances mid-migration), it was never a migration tool.

### Added
- The official Export backup as a mandatory safety step before Remove - the
  export saves INTO the Cloud storage that Remove deletes, so download it first.
- Integration-overwrite recovery: an error page right after connecting is the
  client file being refreshed with a template, not data loss. Includes the
  fix-wiring-only message for the Lovable agent.
- LICENSE file (MIT - the README badge pointed at it, now it exists).

### Removed
- The duplicated `lovable-cloud-supabase-migration-chat/` directory (an exact
  copy of the repo committed by accident).

## v1.0.0 - 2026-05-10

First release. 9 phases, 12 traps from a real end-to-end migration (POMPOM:
13 tables, 175 storage files, 12 edge functions). Backend fully automated from
claude.ai chat via Lovable MCP + Supabase MCP; frontend delegated to Claude Code.
