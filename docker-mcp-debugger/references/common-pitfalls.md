# Common Docker MCP pitfalls (the things that bite everyone)

A short, opinionated tour of the failure modes that are easy to make and hard to spot. If you're stuck and the error code page didn't lead anywhere, skim this — odds are the user did one of these.

## 1. `localhost` inside a container is not the host

The single most common bug. The user runs Obsidian / Postgres / their custom app on their Mac, configures the MCP server with `localhost:<port>`, and gets `Connection refused`.

Inside a Docker container, `localhost` means **the container's own loopback**, not the host machine. Docker Desktop provides a magic DNS name for this: `host.docker.internal`. On Linux, this name doesn't exist by default — Linux users need to pass `--add-host=host.docker.internal:host-gateway` or use the host's LAN IP.

How to spot it: `Connection refused` / `ECONNREFUSED` against a port you know is listening on the host.

Fix: in the server's env or config, change the host string to `host.docker.internal`. Keep the port the same.

Caveat: the target service on the host must be willing to accept connections from the Docker bridge IP, not just from `127.0.0.1`. Some apps bind only to loopback by default and need a setting flipped to listen on `0.0.0.0` or the bridge.

## 2. The custom-server YAML doesn't carry env vars through

When a user adds an MCP server by editing a custom catalog YAML rather than installing from Docker MCP Toolkit's built-in Catalog, the toolkit doesn't run the friendly env-vars form. The user is expected to write the `env:` block themselves. If they forget — or if they put the env vars at the wrong nesting level — the container starts with empty env, the tools register fine, but every call returns 401.

How to spot it: tools list correctly, but a simple tool call returns an auth error.

Fix: open the YAML, locate the server block, ensure `env:` is at the right level (sibling of `image:`, not nested under something weird), and that the key names match exactly what the server image expects. Server-specific names are in `known-servers.md`.

## 3. Two configs trying to do the same thing

Some users end up with the same MCP server defined **both** in Docker MCP Toolkit and directly in `claude_desktop_config.json`. The two paths don't conflict cleanly — sometimes Claude Desktop sees duplicate tools, sometimes it sees one and silently ignores the other.

How to spot it: `claude_desktop_config.json` has an entry for the same server that's also enabled in Docker MCP Toolkit. Or two tools with the same name appear in Claude's tool list.

Fix: pick one path and remove the other. The `docker-mcp` gateway entry in `claude_desktop_config.json` already covers everything in MCP Toolkit, so the per-server entries are redundant.

## 4. Tool list is stale because Claude Desktop wasn't restarted

MCP tool lists are pulled at client startup. Adding a new server and expecting Claude to "notice" without a restart is a recipe for confusion.

How to spot it: the user just added a server, gateway is connected, but tools they expect don't appear in Claude's tool list. They haven't restarted Claude Desktop since adding it.

Fix: fully quit Claude Desktop (Cmd-Q on Mac, not just close the window) and reopen. On the Cowork side, the same goes for whatever client is consuming the MCP — the tool list refreshes on a fresh session, not mid-stream.

## 5. The target app's plugin / API server isn't running

Obsidian's Local REST API is a community plugin that has to be installed *and* enabled. Same for many other apps that expose APIs only via opt-in plugins. The MCP server image assumes that endpoint is up.

How to spot it: Connection refused, but you've already swapped to `host.docker.internal` and the host's port doesn't show up in `lsof -iTCP:<port> -sTCP:LISTEN`.

Fix: walk the user through enabling the plugin / API in the target app. For Obsidian: Settings → Community plugins → enable Local REST API & MCP Server.

## 6. Self-signed certificate, no skip-verify

Obsidian's Local REST API uses HTTPS on 27124 with a self-signed cert by default. Many MCP server images verify TLS strictly out of the box. The connection happens, the handshake fails.

How to spot it: error message contains `self-signed certificate` or `x509` or `unable to verify`.

Fix options (pick one):

- Switch to HTTP on the non-TLS port (27123 for Obsidian) if the target app offers it.
- Set the server's skip-verify flag — for Node-based servers, `NODE_TLS_REJECT_UNAUTHORIZED=0`; for others, check `known-servers.md` for the server-specific knob.
- Trust the cert properly (download from the target app's settings, mount into the container's CA bundle). Cleanest, most effort.

Mention the security tradeoff once when proposing skip-verify, but don't refuse to help — on a single-user local-dev setup talking to `host.docker.internal`, it's a normal choice.

## 7. Secret references that point at nothing

Docker MCP Toolkit supports `${secret.foo}` references in env values. If the secret `foo` was never registered (or was named differently), the variable expands to an empty string and the container starts with a blank credential. Auth errors follow.

How to spot it: 401 errors, env block uses `${secret.…}` syntax.

Fix: in Docker MCP Toolkit, register the secret with the exact name. Or, as a temporary unblock, replace the reference with the literal value (and remind the user to switch back to secrets).

## 8. Port already in use

If the user runs the target app on a non-default port to dodge a conflict, and then leaves the MCP server config on the default, every call goes to the wrong port (or to nothing).

How to spot it: Connection refused, but the target app is definitely running. `lsof -iTCP -sTCP:LISTEN | grep <app>` shows it's on a different port than the config expects.

Fix: align the config's port with the target app's actual port.

## 9. Wrong UID / volume permission

A few MCP servers mount a directory from the host and read/write inside the container. If the container runs as a non-root UID that can't read the mounted directory, you get `Permission denied` on what looks like a perfectly innocent file read.

How to spot it: `Permission denied` errors from a tool that reads files, specifically on files the user can definitely read themselves.

Fix: check the server's docs for which UID it runs as, and adjust permissions on the mounted directory. On macOS, also confirm Docker Desktop has full-disk access in System Settings.

## 10. Docker Desktop / MCP Toolkit version skew

Docker Desktop ships MCP Toolkit as a feature; the toolkit's catalog format has changed between versions. A YAML written against an older version may not load in a newer one (or vice versa). The gateway silently drops servers it can't parse.

How to spot it: server appears in the catalog file but never registers tools; nothing in the gateway logs explains why.

Fix: re-add the server using the current version's UI, which writes the right format. Or compare the user's YAML to a known-good example from a current version's catalog file.

## 11. Looking for a YAML that doesn't exist

In Docker MCP Toolkit (recent versions, 2025+), the **real configuration lives in `~/.docker/mcp/mcp-toolkit.db`** — a SQLite database. The yaml-looking files in that directory (`config.yaml`, `registry.yaml`, `tools.yaml`) are typically 0 bytes — stubs that the toolkit doesn't actually use.

How to spot it: user says "I created a custom yaml to add my server," but `ls -la ~/.docker/mcp/` shows all yaml files at 0 bytes and `mcp-toolkit.db` at multi-MB.

What this means: the user didn't actually write a yaml — they followed the toolkit's first-run setup, which wrote a profile entry to the SQLite db. Common semantic confusion.

Fix: stop looking for a yaml to edit. The only safe path to change a Catalog-installed server's config is the Docker Desktop UI: MCP Toolkit → Profiles → that server → Secrets / config. **Do not hand-edit the SQLite db** — the toolkit owns the schema and corruption is annoying to recover from.

Exception: if the user has a `~/.docker/mcp/catalogs/<name>.yaml` file that is NOT 0 bytes, that's a real custom catalog and can be edited as text.

## 12. The Secret field that looks filled but isn't

Docker MCP Toolkit's Secrets inputs render as `*****` (or similar masked placeholder). **The same visual is shown whether the field is empty (placeholder) or contains a hidden value (masking).** You cannot tell from looking which state you're in.

How to spot it: user reports an auth-shaped error (40101, 401, or an `ENETUNREACH` that's secretly auth — see error-codes.md misleading-errors), and when you ask "did you save the API key in Secrets?", they say "yes, I see asterisks there."

Fix: have the user explicitly do this dance every time, because it's the only way to be sure: click into the field → `Cmd-A` to select whatever's there → `Cmd-V` to paste the actual value → Tab out or click Save → confirm in UI that it persisted. Then restart Claude Desktop so the gateway re-pulls the secret into the container's env.

You can pre-load the value to the user's clipboard with `write_clipboard` so the paste is one keystroke — this also avoids the user having to read a long secret out of one window and into another.

---

## Quick decision tree

```
Tool call errored?
├── Auth-looking (401/403/40101)? → §2, §7, §12  (env / secrets / re-save the field)
├── "Network is unreachable" / ENETUNREACH? → §12 FIRST (empty secret in disguise),
│       then if confirmed-saved → §1 (host.docker.internal) / §5 (service running?)
├── Connection refused?
│     ├── using localhost? → §1  (host.docker.internal)
│     ├── port listening on host? → §5  (target service running?)
│     └── target on non-default port? → §8
├── TLS / certificate? → §6
├── 404 / 405? → check the server's docs for the right URL
├── Tool not found? → §4  (restart Claude Desktop)
└── Permission denied? → §9  (volume / UID)

Tool not even visible?
├── Gateway tools visible? Yes → server didn't register → §2 (malformed yaml IF real yaml exists; see §11), §10 (version skew), or container failed to start
├── Gateway tools NOT visible at all → Docker Desktop not running, MCP Toolkit feature off, or Claude not restarted → §4

User says "I created a yaml" but yaml files are 0 bytes?
└── → §11  (config is in SQLite, fix lives in the UI)
```
