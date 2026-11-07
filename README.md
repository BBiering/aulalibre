# Aula Reversed

Unofficial tools for accessing Denmark's Aula school communication platform. The project delivers three things:

1. **`aula-api`** -- Rust library for the Aula REST API (authentication, models, service calls)
2. **`aula-cli`** -- Command-line interface for reading messages, calendar, posts, gallery, documents, presence, and more
3. **`aula-mcp`** -- MCP server so an LLM client can read and send Aula data in plain language

## CLI commands

Read access to most Aula domains. All commands support `--json` for machine-readable output.

| Command | Description |
|---------|-------------|
| `aula auth login` | Browser-based OIDC login (UniLogin or MitID) |
| `aula auth status` | Show current login state and token expiry |
| `aula auth refresh` | Refresh an expired access token |
| `aula auth logout` | Clear session and tokens |
| `aula messages list` | List message threads |
| `aula messages read <id>` | Read a specific thread |
| `aula messages folders` | List message folders |
| `aula calendar today` | Today's calendar events |
| `aula calendar week` | This week's events |
| `aula calendar show <id>` | Show a specific event |
| `aula presence status` | Children's current presence |
| `aula presence schedule` | This week's presence schedule |
| `aula posts list` | List institution feed posts |
| `aula posts show <id>` | Show a specific post |
| `aula gallery list` | List photo albums |
| `aula gallery show <id>` | Show media in an album |
| `aula documents list` | List secure documents |
| `aula documents show <id>` | Show a specific document |
| `aula notifications list` | List recent notifications |
| `aula search <query>` | Search profiles and groups |
| `aula groups list` | List groups for an institution profile |
| `aula groups show <id>` | Show group details |
| `aula groups members <id>` | Show group members |
| `aula profile me` | Show your profile |
| `aula profile master-data` | Show name, email, phone |
| `aula config ...` | View and manage CLI configuration |

Global flags: `--json`, `--env <environment>`, `--verbose`, `--profile <name>`.

## MCP server

`aula-mcp` exposes Aula to any MCP client over stdio, so you can ask questions instead of memorising CLI flags.

```bash
nix-shell
just build-release
# point your MCP client at the binary:
#   {"mcpServers": {"aula": {"command": "<repo>/aula/target/release/aula-mcp"}}}
```

Authentication is shared with the CLI -- run `just run auth login` once and the server picks up the same token store. Set `AULA_ENV` to target a non-production environment. When the login eventually expires, see [Refreshing the login](#refreshing-the-login).

Tools are read-oriented (`aula_whoami`, `aula_messages`, `aula_thread`, `aula_calendar`, `aula_week_plan`, `aula_presence`, `aula_posts`, `aula_notifications`, `aula_gallery`, `aula_documents`, `aula_search`) plus two writes: `aula_send_message` and `aula_reply`. Responses are compact projections of Aula's DTOs -- plain text instead of HTML, only the fields a question can be answered from.

To use it from Claude mobile, the server has to run remotely -- see [`re/remote_mcp_plan.md`](re/remote_mcp_plan.md).

## Getting started

Prerequisites: [Nix package manager](https://nixos.org/download.html) and a browser for login.

```bash
nix-shell               # enter dev environment
just build              # build everything
just run auth login --manual  # authenticate via browser
just run messages list        # list message threads
```

## Configuration

`~/.config/aula/config.toml`:

```toml
default_environment = "production"
default_format = "text"
default_profile = "MyChild"
verbose = false
```

All settings can be overridden with CLI flags.

## Authentication

OIDC Authorization Code + PKCE flow through `login.aula.dk`.

- **Manual** (`--manual`, recommended): opens browser, you paste the redirect URL back into the terminal.
- **Automatic** (default): opens browser and captures the auth code via a local callback server. May not work in all environments.

Tokens are stored in the platform data directory with `0600` permissions:
`~/Library/Application Support/aula/tokens.json` on macOS, `$XDG_DATA_HOME/aula/tokens.json`
(or `~/.local/share/aula/`) on Linux. `AULA_DATA_DIR` overrides the location.

### Refreshing the login

The access token lasts about an hour and refreshes itself — normal use needs no
intervention. What eventually expires is the **refresh token**, and then every call fails:
the CLI returns an API error, and the hosted server says `not logged in`.

**CLI only:**

```bash
just run auth refresh     # refresh in place, keeps the current session
just run auth login       # full re-login with MitID — needed once the refresh token dies
```

**Hosted MCP server** (the one in Claude): it reads its tokens from Supabase, so a local
re-login alone is not enough — push them across:

```bash
just run auth login                     # MitID; writes the local token file
set -a; . secrets/supabase.env; set +a  # SUPABASE_URL + SUPABASE_SECRET_KEY
just push-tokens                        # upserts the token row, prints the new expiry
just verify-deployment                  # needs AULA_MCP_URL + AULA_MCP_SECRET
```

No redeploy is needed: the server reads the row on every invocation, and the
`aula_whoami` line of `verify-deployment` proves it picked up the new tokens. If that line
says `not logged in`, the row is still empty — `push-tokens` did not run against the right
project (check `secrets/supabase.env`).

## Project structure

```
aula/                    Rust workspace
  aula-api/              API client library (auth, models, services)
  aula-cli/              CLI binary (clap, one module per domain)
  aula-mcp/              MCP server binary (stdio JSON-RPC, tool projections)
re/                      Reverse engineering analysis documents
secrets/                 Gitignored: dev artifacts, HAR dumps, test fixtures
scripts/                 Helper scripts (login, chrome devtools)
shell.nix                Nix environment
Justfile                 Common recipes (build, test, lint, run, mcp)
```

## License

MIT License. See [LICENSE](LICENSE).
