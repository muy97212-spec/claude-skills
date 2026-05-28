# Per-server quirks

Specific MCP server images have specific gotchas — env var names, default ports, plugin prerequisites, TLS choices. Use this when the user names a server and you want to skip the slow generic diagnosis. If the user's server isn't here, fall back to the five-layer method in `SKILL.md`.

## mcp-obsidian

- **Target service**: Obsidian "Local REST API & MCP Server" community plugin.
- **Plugin must be installed and enabled in Obsidian**, AND Obsidian must be running. The plugin shows the API key in its settings panel — that's the value the user needs.
- **Two deployment routes** — see "Direct route alternative" below; choose one, delete the other to avoid duplicate tool registration.

### Route 1: Through Docker MCP Toolkit (MarkusPfundstein container)

- **Env vars (typical names)**:
  - `OBSIDIAN_API_KEY` — required; paste from the plugin's settings page. **In Docker MCP Toolkit this is set via the Secrets field, not by editing a file** — see common-pitfalls.md §11 / §12.
  - `OBSIDIAN_HOST` — set to `host.docker.internal` on Mac/Win; leave as `localhost` only if the MCP server is running outside Docker.
  - `OBSIDIAN_PORT` — defaults to 27124 (HTTPS). If using HTTP, set 27123.
  - TLS skip-verify: depends on the image. Common is `OBSIDIAN_INSECURE_TLS=true` or `NODE_TLS_REJECT_UNAUTHORIZED=0`.
- **Canonical errors**:
  - `Error 40101: Authorization required.` → missing or wrong `OBSIDIAN_API_KEY`. Re-paste in the Secrets field (see §12).
  - **`Network is unreachable to host.docker.internal:27124` / `Errno 101`** → **first suspect empty Secret, not routing.** This container has a confirmed pattern of throwing ENETUNREACH when the API key env var is an empty string. See error-codes.md → Network-family → "Critical caveat for ENETUNREACH" for the two-step disambiguation.
  - `Connection refused` (real, after secret confirmed-saved) → either `localhost` instead of `host.docker.internal`, or Obsidian/plugin not running.
  - `self-signed certificate` → either flip to HTTP on 27123, or set the skip-verify flag.
- **Tools** (12 total via this route): `list_files_in_vault`, `simple_search`, `complex_search`, `get_file_contents`, `batch_get_file_contents`, `list_files_in_dir`, `get_recent_changes`, `get_recent_periodic_notes`, `get_periodic_note`, `append_content`, `patch_content`, `delete_file`. `list_files_in_vault` is the safest one to call for a diagnostic probe — no input, no side effects.

### Route 2: Direct via `mcp-remote` (no Docker)

The "Local REST API & MCP Server" plugin (2025+) **is itself an MCP server** at `https://127.0.0.1:27124/mcp/` (or `http://127.0.0.1:27123/mcp/` if HTTP server is enabled in the plugin settings). You can wire Claude Desktop directly to it via a one-line proxy entry in `claude_desktop_config.json`:

```json
"obsidian": {
  "command": "npx",
  "args": [
    "-y",
    "mcp-remote",
    "http://127.0.0.1:27123/mcp/",
    "--header",
    "Authorization: Bearer <THE_API_KEY>"
  ]
}
```

For HTTPS (27124), add `"env": {"NODE_TLS_REJECT_UNAUTHORIZED": "0"}` because of the self-signed cert. HTTP-on-localhost is simpler and equally safe on a single-user dev machine.

Tools registered via this route get the prefix `mcp__obsidian__*` (versus `mcp__MCP_DOCKER__obsidian_*` for the Docker route). Different names, no actual collision, but having both registered means duplicates in the tool picker.

### When to pick which

- **Docker route**: better when the user is already managing many MCP servers through Docker MCP Toolkit's UI and wants one place to see them all.
- **Direct route**: better when Obsidian is the only MCP server the user needs, or when debugging the Docker route has stalled. Fewer moving parts: Claude → mcp-remote → plugin.

## mcp-github

- **Target service**: GitHub REST/GraphQL API (no local component).
- **Env vars**:
  - `GITHUB_PERSONAL_ACCESS_TOKEN` — classic PAT or fine-grained token.
- **Network layer is just GitHub.com** — `host.docker.internal` doesn't apply. If the container has no outbound internet, all calls fail.
- **Canonical errors**:
  - `401 Bad credentials` → token wrong, expired, or revoked.
  - `403 Forbidden` with rate-limit headers → hit the unauthenticated or low-scope rate limit.
  - `403` on a specific repo → token doesn't have scope for that repo or that action (e.g. fine-grained token without write permission).
- **Diagnostic probe**: list repositories for the authenticated user, or fetch user info. Doesn't change anything.
- **Common owngoal**: pasting a token that starts with `ghp_` but truncating it. Tokens are long; YAML line-wrapping mangles them.

## mcp-slack

- **Target service**: Slack Web API.
- **Env vars**:
  - `SLACK_BOT_TOKEN` (starts with `xoxb-`) — most common.
  - `SLACK_TEAM_ID` — required by some images.
- **Canonical errors**:
  - `invalid_auth` → wrong token.
  - `not_authed` → no token passed.
  - `missing_scope` → token works but lacks the scope for the call.
- **Diagnostic probe**: list channels.
- **Trap**: confusing user token (`xoxp-`) with bot token (`xoxb-`). Most MCP servers want bot.

## mcp-postgres / mcp-mysql

- **Target service**: user's local or remote database.
- **Env vars** (names vary by image, but typically):
  - `POSTGRES_HOST` / `MYSQL_HOST` — `host.docker.internal` for a DB on the same Mac/Win machine.
  - `POSTGRES_PORT` / `MYSQL_PORT` — 5432 / 3306 by default.
  - `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`.
- **Canonical errors**:
  - `Connection refused` → §1 from `common-pitfalls.md` (localhost vs host.docker.internal), or DB not running.
  - `password authentication failed for user "x"` → wrong password.
  - `database "x" does not exist` → wrong DB name in config.
- **Listen-address gotcha**: a freshly-installed Postgres often binds only to `127.0.0.1`. To accept the Docker bridge, edit `postgresql.conf` (`listen_addresses = '*'`) and `pg_hba.conf` (add a line for the docker bridge subnet).

## mcp-notion

- **Target service**: Notion API.
- **Env vars**: `NOTION_API_KEY` (integration token from Notion's developer settings).
- **Trap**: a Notion integration sees nothing unless the user has explicitly shared pages/databases with the integration in Notion's UI. The MCP server can authenticate fine and still get empty results — that's not a bug, the user just hasn't shared anything.
- **Diagnostic probe**: search for any page. Empty list ≠ broken; ask the user if they shared anything with the integration.

## mcp-filesystem

- **Target service**: a directory on the host, exposed as MCP tools.
- **Volume mount required** — the YAML must have a `volumes:` block mapping a host directory into the container.
- **Canonical errors**:
  - `Permission denied` → see §9 in `common-pitfalls.md`.
  - `No such file or directory` for the root path → the mount didn't happen.
- **Trap**: macOS Docker Desktop won't bind-mount directories outside the user's allowed list. Check Docker Desktop → Settings → Resources → File sharing.

## mcp-google-drive / mcp-gmail / mcp-google-calendar

- **Target service**: Google APIs.
- **OAuth-based**, not just an API key. The MCP server usually needs a path to an OAuth credentials JSON, or environment vars for client_id / client_secret / refresh_token.
- **Canonical errors**:
  - `invalid_grant` → refresh token expired or revoked; need to redo OAuth flow.
  - `insufficient_scope` → the OAuth consent didn't grant the right scopes; redo with the scopes the server requires.
- **Trap**: credentials JSON mounted at a path the container can't read (§9).

## How to add a new server to this file

When you debug a server that isn't listed here, write a short entry: target service, env vars by exact name, canonical errors with their resolutions, a safe diagnostic probe. Keep the format tight — this file is meant to be skimmed, not read end-to-end.
