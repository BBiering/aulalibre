# Session summary — 2026-11-07

A single working session that started with a broken FUSE mount and ended with a hosted MCP
server used daily from Claude Desktop and mobile. This records what changed, and more
usefully, the dead ends — most of the effort went into diagnosing things that were not in
the plan.

Base: `ed41606` (upstream `main`). Result: 27 commits, 44 files, +6223/−2628.

## Starting point

`just mount` failed. The repo had a Rust FUSE filesystem (`aula-fuse`) that could not run:
no FUSE runtime was installed, and after installing the two candidates, both were broken on
macOS 26 — fuse-t's NFS backend panics inside go-nfsv4, its FSKit bundle is absent from the
1.2.9 package, and macFUSE needs a kernel extension plus a Reduced Security boot policy.

The pivot: drop FUSE, expose Aula as an **MCP server** instead, so an LLM can be asked
questions rather than a filesystem being browsed.

## What was built

1. **`aula-mcp`** — MCP over stdio: `initialize`, `notifications/*`, `ping`, `tools/list`,
   `tools/call`, 13 tools (11 read, 2 write). Responses are compact projections of Aula's
   DTOs (HTML stripped, bodies truncated, nulls dropped) rather than raw models.
2. **`aula-vercel`** — the same dispatcher over HTTP for remote clients: `api/mcp.rs` plus
   `api/healthz.rs`, with the secret accepted from `x-api-key`, `authorization` or a query
   parameter.
3. **Token storage** — `TokenStore` gained a `Memory` backend and `SupabaseTokens` reads and
   writes a Postgres row through PostgREST. Hosted invocations seed a session from the row
   and write back only when the session refreshed.
4. **`aula_week_plan`** — Aula has no week-plan endpoint; schools embed one from a third
   party. Meebook, reached through an Aula widget SSO token, backs the tool.
5. **Ops** — `scripts/push_tokens.sh`, `scripts/verify_deployment.sh`, a daily cron that
   keeps the free-tier database awake, and `re/vercel_deploy.md`.
6. **Removed** — `aula-fuse` (~2k lines), its recipes, the `fuse3` build input.

## Bugs found, and their real causes

- **Aula retired API v23.** Every version except v25 answers `410 Gone`. The version was
  hardcoded in three places; now one `DEFAULT_API_VERSION`.
- **A refresh that never ran.** `ensure_context_initialized()` calls `client.get()` directly
  to avoid recursion, so it bypassed `pre_request()` and sent expired tokens as-is. Aula
  answers those with `403 / code 20` — indistinguishable from a permission error, which is
  why it read as "log in again". Refresh first.
- **A refresh race.** Two clients sharing one token store both refresh on expiry; the loser
  presents a rotated-away token and stays broken until restart. It now adopts whatever the
  winner persisted.
- **Notifications answered 204.** The Streamable HTTP transport requires 202 with no body;
  204 makes strict clients treat the exchange as failed. This sat on the connector handshake.
- **Deploys silently went to preview.** Production branch tracking did not point at the
  pushed branch, so the production domain kept serving the old build. Two rounds were lost
  to this; it is now the first thing the runbook tells you to check.

## Dead ends worth not repeating

| Attempt | Why it failed |
|---|---|
| fuse-t (NFS backend) | go-nfsv4 panics on the first client request |
| fuse-t (FSKit backend) | `/Library/Filesystems/fuset.fs` not shipped in 1.2.9 |
| macFUSE kext | needs Reduced Security + approval + reboot |
| `X-Aula-Secret` header | Anthropic rejects unapproved header names; only `authorization`, `x-api-key`, `x-auth-token` work |
| `tools/list` as a health check | passes on a hollow deployment; only a real tool call proves tokens and Aula |
| A grep-based PII scan | `git grep` rejected the argument order and returned zero hits silently; a control term caught it |
| EasyIQ / Systematic MOMO | no widget token / 403 for this account |
| Min Uddannelse opgaver+ugebrev | endpoints answer 200 but return empty arrays; needs its own redirect/`elevId` flow |

## Cost

$0/month. Free-tier allowances are ~3 orders of magnitude above usage: at 1,000 tool calls a
month the function uses ~1.4% of Hobby's 4 active-CPU hours, and the database row is 2 KB
against a 500 MB allowance.

## Deliberately not built

- **Re-login page** — `just push-tokens` covers the case today; build it when the refresh
  token actually expires.
- **Onboarding cache** — measured at ~1.1 s per call, judged acceptable for interactive use.
- **Multi-user support** — would need per-user tokens and OAuth 2.1, which is a different
  design rather than a tweak.

## Where to read further

| Document | Contents |
|---|---|
| `README.md` | CLI and MCP usage, refreshing the login, connector setup |
| `re/remote_mcp_plan.md` | design rationale, R1–R8 status, provider findings |
| `re/vercel_deploy.md` | deployment runbook: env vars, region, protection trap, verification |
| `re/supabase_schema.sql` | token table, RLS, `updated_at` trigger |
| `aula-api/tests/widget_probe_live_tests.rs` | reproducible probe of the third-party providers |

## Verification at the end of the session

- `cargo test --workspace` green across 8 suites (533 test functions), clippy `-D warnings`
  clean, `just e2e` clean.
- Live: every read tool against a real account; auto-refresh forced on both the file store
  and the Supabase row; the deployed endpoint end to end.
- `just verify-deployment`: healthz 200, no-secret 404 (Deployment Protection off), wrong
  secret 404, `GET` 405, 13 tools, real `aula_whoami` in 0.8–1.1 s.
- Connector confirmed working from Claude Desktop and mobile.
