---
name: docker-mcp-debugger
description: Diagnose and fix Docker-based MCP servers that aren't working — tools missing from Claude, 401/40101 auth errors, "Connection refused" or "Network is unreachable"/ENETUNREACH from the container, self-signed cert failures, Secrets that look filled in Docker MCP Toolkit's UI but aren't actually saved, host.docker.internal vs localhost mistakes, or servers that connect but won't run. Use whenever the user reports a Dockerized MCP server is broken, missing, or returning errors — even when they only say "my MCP isn't working" without naming Docker. Also trigger when they mention Docker MCP Toolkit profiles, the Secrets field showing asterisks, custom catalog YAML, mcp-obsidian / mcp-github / mcp-slack / mcp-* containers, mcp-remote as a direct alternative, or a tool list missing entries they expected. The skill maps error codes to stack layers, including misleading patterns (e.g. ENETUNREACH that's secretly an empty Secret), and patches editable surfaces or hands tight instructions for the ones it can't touch.
---

# Docker MCP Debugger

## What this skill is for

Docker-based MCP servers (whether installed via Docker MCP Toolkit, run by hand with `docker run`, or wired into Claude Desktop's `claude_desktop_config.json`) fail in a small number of repeating ways. The user usually sees a vague symptom — "tools don't show", "it's broken", "Claude says it can't reach it" — but the actual fault is in one specific layer of a multi-layer stack. Guessing wastes time; reading the error message wastes none.

The strategy this skill follows is:

1. **Provoke a precise error** by actually calling the simplest tool on the suspect server.
2. **Map the error to the layer that produced it** (gateway, container runtime, container→host network, target service, auth).
3. **Patch the config** if the fix is mechanical (missing env var, wrong host string, missing flag) — or **give the user a tight, scoped instruction** if it requires action outside Claude's reach (enabling a plugin, restarting Claude Desktop, regenerating an API key).
4. **Verify** by re-calling the same tool.

The reason this works: every layer of the stack returns a recognizable error code or message when it's the one that broke, and you almost never see ambiguous overlaps if you actually look at the error.

## The five-layer model

When debugging, always think in terms of these layers. The error code tells you which one is bad.

```
┌─────────────────────────────────────────────────────────┐
│  L1  Claude client (Desktop, Cowork, Code) ←──┐         │
├─────────────────────────────────────────────────────────┤
│  L2  Docker MCP Gateway (the docker-mcp aggregator)     │
├─────────────────────────────────────────────────────────┤
│  L3  Individual MCP server container (mcp-obsidian etc) │
├─────────────────────────────────────────────────────────┤
│  L4  Container → host network (host.docker.internal)    │
├─────────────────────────────────────────────────────────┤
│  L5  Target service on host (Obsidian REST API, etc.)   │
└─────────────────────────────────────────────────────────┘
```

A failure at layer N looks different from a failure at layer N±1. The diagnostic question at each step is just "which layer answered this error?"

## Diagnostic workflow

Follow these steps in order. Don't skip ahead to fixes.

### Step 1 — Confirm L1↔L2: is the gateway visible at all?

Check whether *any* `mcp__MCP_DOCKER__*` (or whatever the user's gateway prefix is) tools are loaded. If the user is in Cowork/Code, they may be deferred — use `ToolSearch` with `query: "select:<some_known_tool>"` to load one and see if it appears.

- **If zero gateway tools exist anywhere**: L1↔L2 is broken. The gateway itself didn't connect. Common causes: Docker Desktop not running, MCP Toolkit feature not enabled, Claude Desktop wasn't restarted after install. **Don't proceed** until this is fixed; nothing downstream can work.
- **If gateway tools exist**: L1↔L2 is fine. Move on.

### Step 2 — Confirm L2↔L3: is the specific server's tools visible?

Ask: "Of the gateway tools, are any from the server the user is asking about?" For example, looking for `*obsidian*`, `*github*`, `*slack*` in the tool list.

- **Tools from that server are missing**: the gateway connected, but **this specific server didn't register**. Possible causes:
  - The server isn't enabled in the active profile (Docker MCP Toolkit → Profiles → make sure the server has the toggle on)
  - The custom server YAML/JSON the user wrote is malformed and the gateway silently dropped it
  - The server image failed to start (check `docker ps -a` — see if the container exists and what its exit code was)
  - Claude Desktop was opened before the server was added, so the tool list is stale. **Tool lists are pulled at client start** — a restart is required after adding servers.
- **Tools from that server are present**: L2↔L3 is fine. Move on to Step 3.

### Step 3 — Provoke a precise error

Pick the simplest, lowest-side-effect tool from that server (e.g. `list_files_in_vault`, `list_repositories`, `list_channels`) and **actually invoke it**. Don't reason about what might be wrong — call it and read the error verbatim.

The error you get is the most valuable single piece of diagnostic information. Most error messages are scoped to a single layer.

### Step 4 — Map the error to a layer

Read `references/error-codes.md` for the full table. The short version:

| Error contains… | Most likely layer | Fix family |
|---|---|---|
| `Authorization`, `401`, `403`, `40101`, `Invalid API key` | L5 (target service auth) | Missing/wrong env var passed to L3 |
| `Connection refused`, `ECONNREFUSED`, `connect: cannot assign requested address` | L4 (container→host network) | `localhost` should be `host.docker.internal` (Mac/Win) |
| `Network is unreachable`, `ENETUNREACH`, `Errno 101` | **First L5 (auth)**, then L4 | **Verify Secret is actually saved before chasing routing** — see below |
| `Timeout`, `ETIMEDOUT`, hangs and never returns | L4 or L5 | Wrong port, firewall, or target service hung |
| `self-signed certificate`, `unable to verify the first certificate`, `x509` | L4/L5 (TLS) | Add insecure/skip-verify flag, or trust the cert |
| `404 Not Found`, `no such endpoint` | L5 config | Wrong base URL / path / API version |
| `Tool not found`, `unknown tool` | L2/L3 protocol | Stale tool list — restart Claude client |
| `Permission denied` opening a file/socket | L3 container perms | Volume mount missing or wrong, or running as wrong UID |

When in doubt, read the **entire** error including stack traces — server names and ports inside the message often confirm the layer.

**⚠️ The ENETUNREACH-as-auth trap.** Some MCP server images (confirmed on `mcp-obsidian`) will throw `Network is unreachable` / `Errno 101` when the secret they need is an empty string, not just when the network is actually broken. Mechanism: the server's HTTP client tries to construct an `Authorization: Bearer ` header with an empty token, hits an internal exception during socket setup, and that exception bubbles up wearing the costume of a routing failure. So when you see `Network is unreachable` on a Dockerized MCP server, **verify the secret is actually saved first** (see common-pitfalls.md §11 "Secret field looks filled but is empty"), and only if that's confirmed-good should you start investigating `host.docker.internal` / IPv6 / extra_hosts.

### Step 5 — Apply the fix

The fix surface depends on **how the server was installed**. There are three real options, and they have different editability — the skill should know which one is in play before proposing edits.

**Surface A — Docker MCP Toolkit, server from the official Catalog** (most common in 2026)
- Config lives in `~/.docker/mcp/mcp-toolkit.db` (a SQLite database).
- The yaml stubs in `~/.docker/mcp/` (`config.yaml`, `registry.yaml`, `tools.yaml`) are **typically 0 bytes** — the toolkit doesn't actually use them as files; the UI writes to the SQLite db. **Editing those yaml files does nothing.**
- The only safe way to change env / secrets is through the Docker Desktop UI: MCP Toolkit → Profiles → that server → Secrets / config block.
- **The skill cannot do this for the user via computer-use** (Docker Desktop is often not in the OS-level allowlist). Give the user a precise click-by-click instruction and offer to put values on their clipboard.

**Surface B — Docker MCP Toolkit, custom catalog YAML** (less common)
- If the user has files in `~/.docker/mcp/catalogs/` that are NOT empty, those are real custom server definitions and *can* be edited as text.
- Same caveats apply: changes only take effect after Docker MCP Toolkit reloads / Claude Desktop restarts.

**Surface C — `claude_desktop_config.json` direct entry** (no Docker)
- `mcpServers.<name>.command` is the full path / shell command. The skill can edit this file directly with `Edit` / `Write` (TextEdit GUI not required if the harness has access).
- This is also the place to wire up an HTTP-based MCP server via `npx mcp-remote <url> --header "Authorization: Bearer ..."` — see "Two deployment modes" below.

For all three, **mechanical fixes** (missing env var, wrong host string, missing TLS flag) can be applied with the user's confirmation in the appropriate surface. **Non-mechanical fixes** (enable a plugin, restart Claude, regenerate API key) need a tight numbered list for the user to execute themselves.

Read `references/common-pitfalls.md` for the canonical fixes per pitfall.

## Two deployment modes (when to skip Docker entirely)

For a target service whose own app already speaks MCP natively (e.g. recent Obsidian "Local REST API & MCP Server" plugin exposes `http://127.0.0.1:27123/mcp/` directly), you have two ways to wire it into Claude Desktop:

1. **Through Docker MCP Toolkit's gateway** — install a third-party container (`mcp-obsidian` etc.) that translates between MCP protocol and the target's REST API. More moving parts: Claude → gateway → container → host network → target. Five potential failure points.
2. **Direct via `npx mcp-remote`** — a one-line entry in `claude_desktop_config.json` that bridges the target's HTTP/SSE MCP endpoint into Claude Desktop's stdio expectation. Two moving parts: Claude → mcp-remote → target. Almost nothing to break.

When the target supports both, **mention both options to the user** so they can choose. The Docker route is preferable when they're already managing many servers through MCP Toolkit's UI; the direct route is preferable when this is the only server or when debugging the Docker route has stalled. Whichever route is chosen, **delete the other one** to avoid two providers registering tools with overlapping names (the prefixes differ — `mcp__obsidian__*` vs `mcp__MCP_DOCKER__obsidian_*` — but having two of the same vault tool in the picker is just noise).

### Step 6 — Verify

Re-invoke the same tool you used in Step 3. The error should either be gone, or have moved to a different layer (which is progress — repeat from Step 4 with the new error). Don't declare victory based on "the config looks right now"; declare victory based on a green tool call.

## Auto-fix scope (what this skill will and won't touch on its own)

**Will edit, with explicit user confirmation each time:**

- `claude_desktop_config.json` — add/remove `mcpServers.*` entries, edit env blocks, add `mcp-remote` proxy entries
- A real (non-empty) custom-catalog YAML in `~/.docker/mcp/catalogs/`
- Swap `localhost` / `127.0.0.1` → `host.docker.internal` in any editable config
- Add an `--insecure` / `NODE_TLS_REJECT_UNAUTHORIZED=0` flag where the user has accepted the security tradeoff
- Fix obvious YAML/JSON syntax errors that prevent the gateway from loading the server

**Will not do automatically — instruct the user instead:**

- **Anything inside Docker Desktop's UI** (Secrets fields, profile toggles, server install/remove) — the OS-level app allowlist usually doesn't include Docker Desktop, so computer-use can't reach it. Hand the user a tight numbered list and pre-load values to their clipboard via `write_clipboard` when an API key is needed.
- **Edit `~/.docker/mcp/mcp-toolkit.db`** — it's SQLite and the toolkit owns the schema. Hand-editing risks corruption.
- Restart Claude Desktop (the harness lives inside Claude Desktop; quitting kills the session). Hand it off.
- Enable a community plugin inside Obsidian / Notion / etc.
- Generate a new API key inside a third-party app
- Move money, execute trades, or change anything that has side effects beyond MCP wiring
- Edit configs that look unrelated to the reported problem (no drive-by changes)

**Never:**

- Commit secrets to a file the user didn't explicitly tell you about
- Write a real API key into a chat log or any file outside the user's config path
- Disable security checks (TLS verification, etc.) without surfacing the tradeoff first

## Locating the config files

Mac (most common):

- **Claude Desktop config**: `~/Library/Application Support/Claude/claude_desktop_config.json` — editable JSON. This is where `mcpServers.*` entries live, including any `mcp-remote` direct proxies.
- **Docker MCP Toolkit data**: `~/.docker/mcp/`
  - `mcp-toolkit.db` — **SQLite, holds the real configuration** (profiles, enabled servers, secrets, settings). Do NOT hand-edit. All changes go through Docker Desktop UI.
  - `config.yaml`, `registry.yaml`, `tools.yaml` — usually 0 bytes in recent versions. Stubs. Don't waste time on these.
  - `catalogs/` — only present if the user has actually written a custom catalog. Most users don't have this directory at all.

Windows: `%APPDATA%\Claude\claude_desktop_config.json`, `%USERPROFILE%\.docker\mcp\`
Linux: `~/.config/Claude/claude_desktop_config.json`, `~/.docker/mcp/`

**Important meta-point about the yaml stubs.** When a user says "I created a YAML to add my server" but the `~/.docker/mcp/*.yaml` files are 0 bytes, they almost certainly didn't actually write a YAML — they followed the toolkit's first-run setup which writes a profile to the SQLite db. This is a common semantic confusion worth gently clarifying so you don't waste time looking for a yaml that doesn't exist.

If the paths don't exist, ask the user — different Docker Desktop versions move things and custom installs vary. **Always confirm the file path before editing.**

## Per-server notes

For known servers with quirks (default ports, env var names, certificate behavior), read `references/known-servers.md`. Common entries: `mcp-obsidian`, `mcp-github`, `mcp-slack`, `mcp-notion`, `mcp-postgres`, `mcp-filesystem`. If the user's server isn't listed and the error is unfamiliar, fall back to the generic five-layer method.

## Anti-patterns to avoid

- **Guessing without calling a tool first.** The error code is free information. Use it.
- **"Try restarting Claude" as a first move.** It does sometimes help (stale tool list), but if you reach for it before reading an error, you'll restart five times and learn nothing.
- **Adding `--insecure` to silence a TLS error without explaining the tradeoff.** It's a valid fix in many local-dev cases, but the user should know what was traded.
- **Editing the wrong config file.** Docker MCP Toolkit configs and Claude Desktop's `claude_desktop_config.json` are *different files*. A server added via MCP Toolkit may not appear in `claude_desktop_config.json` at all — it's surfaced through the `docker-mcp` gateway entry there. Don't add duplicate entries.
- **Hiding the secret-handling issue.** If a user has been pasting raw API keys into config files, mention (briefly, once) that Docker MCP Toolkit's secrets mechanism is the safer pattern — but don't refuse to help them get unblocked first.

## Example trace (the canonical Obsidian case, from a real session)

Symptom: "I added Obsidian to Docker MCP Toolkit. Claude Desktop sees docker-mcp fine, the obsidian_* tools show up, but calling them errors out."

1. **L1↔L2 OK** — `mcp__MCP_DOCKER__*` tools exist.
2. **L2↔L3 OK** — `mcp__MCP_DOCKER__obsidian_*` tools exist (gateway registered the server).
3. **Provoke** — call `obsidian_list_files_in_vault`. Error: `Error 40101: Authorization required.`
4. **Map** — 40101 is L5 (Obsidian Local REST API plugin) rejecting the credential. The credential is missing or empty.
5. **Investigate where config actually lives** — user said "I made a custom yaml," but `ls -la ~/.docker/mcp/` shows all yaml stubs are 0 bytes and `mcp-toolkit.db` is 5 MB. **Config is in SQLite, edited via UI.** No yaml to edit. Reframe the fix.
6. **First fix attempt** — user opens Docker Desktop → MCP Toolkit → Profiles → Obsidian, sees `obsidian.api_key` field showing `*****`. They re-paste the key. Restart Claude Desktop. Re-call the tool.
7. **New error** — `Network is unreachable to host.docker.internal:27124`. Looks like L4. Tempting to chase host.docker.internal / IPv6 / extra_hosts.
8. **Stop. Apply the ENETUNREACH-as-auth check** — `curl -k https://localhost:27124/` from the host returns JSON, so the target service is fine on L5. That makes a true L4 routing failure unlikely (Docker Desktop almost always wires host.docker.internal correctly on Mac). Hypothesis: the Secret didn't actually save in step 6 — the `*****` was a placeholder both before AND after, the field looks the same in both states.
9. **Real fix** — user clicks back into the secret field, Cmd-A to select existing (placeholder or whatever's there), Cmd-V to paste, makes sure to Tab out / hit Save. Restart Claude Desktop.
10. **Verify** — `obsidian_list_files_in_vault` returns the vault file list. Done.

**Lessons baked into the rest of this skill:**
- Don't assume edit surface is a text file when Docker MCP Toolkit is in play. Check whether the yaml stubs are 0 bytes first.
- Don't trust the visual state of a Secret field. If it shows `*****`, that's either a placeholder OR a hidden value — you can't tell. Always have the user re-paste and confirm save.
- Don't chase L4 routing on `Network is unreachable` until you've ruled out L5 secret-empty.

That's the whole loop.
