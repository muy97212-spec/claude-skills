# Error code → layer → fix reference

Use this when an MCP tool call returned an error and you need to know which layer of the stack to look at. Errors are grouped by the layer they typically come from.

## Table of contents

- Auth-family errors (L5)
- Network-family errors (L4)
- TLS / certificate errors (L4 / L5)
- HTTP-shape errors (L5)
- Protocol / tool-listing errors (L2 / L3)
- Container / filesystem errors (L3)
- Misleading errors (when the visible layer isn't the broken one)

---

## Auth-family errors → L5 (target service is rejecting the credential)

Hallmarks: HTTP-status-style numbers, the word "Authorization" or "Unauthorized" or "Forbidden", an API-specific error code that the target service docs would recognize.

Examples seen in the wild:

- `Error 40101: Authorization required.` — Obsidian Local REST API plugin
- `401 Unauthorized` — most REST APIs
- `403 Forbidden` with a body like `{"error":"invalid_token"}` — GitHub, Slack
- `Invalid API key` / `Bad credentials`

What this means: the MCP server inside L3 **successfully reached the target service** at L5, opened a network connection, and was rejected at the application layer. The container is fine. The network is fine. The credential isn't.

What to check, in order:

1. The `env:` block in the server's definition. Is the expected env var name actually present? (Each server uses a specific name — `OBSIDIAN_API_KEY`, `GITHUB_PERSONAL_ACCESS_TOKEN`, `SLACK_BOT_TOKEN`, etc. See `known-servers.md`.)
2. If the value is a secret reference (e.g. `${secret.foo}`), is that secret actually registered in Docker MCP Toolkit's secret store?
3. Is the value's leading/trailing whitespace clean? Multi-line YAML strings sometimes pick up newlines.
4. Did the user regenerate the key recently in the target app and forget to update the config?
5. For tokens with scopes (GitHub, Slack), is the scope sufficient for the operation that failed? Some servers only fail per-call, not at startup.

---

## Network-family errors → L4 (usually) or L5-with-empty-secret (sometimes)

Hallmarks: `ECONNREFUSED`, `Connection refused`, `getaddrinfo ENOTFOUND`, `cannot assign requested address`, `network unreachable`, `Network is unreachable`, `EHOSTUNREACH`, `ENETUNREACH`, `Errno 101`.

What this usually means: the MCP server inside the container tried to open a TCP connection to a host:port and the OS refused or couldn't route. The credential never got a chance to be checked.

**⚠️ Critical caveat for `ENETUNREACH` / `Network is unreachable` / `Errno 101`.** Some MCP server images (confirmed on `mcp-obsidian` MarkusPfundstein build) raise this error when their **secret env var is an empty string**, not just when routing actually fails. The internal HTTP client tries to build an `Authorization: Bearer ` header, hits an exception in socket setup, and that exception bubbles up looking exactly like a routing failure. So when you see ENETUNREACH on a Dockerized MCP server, **before chasing host.docker.internal / IPv6 / extra_hosts**, do this two-step check:

1. **From the host**, can you `curl` the target service? e.g. `curl -k https://localhost:27124/`. If yes, the service is up — host-side L5 is fine.
2. **Is the relevant Secret actually filled in Docker MCP Toolkit's UI?** Have the user re-paste the value into the Secret field and explicitly save, because the UI may show `*****` indistinguishably whether the field is empty (placeholder) or filled (masked value).

If both check out, *then* it's likely a real routing problem and the rest of this section applies.

**Most common true-L4 cause on Mac/Windows**: the server config uses `localhost` or `127.0.0.1`. Inside a Docker container, those refer to **the container itself**, not the host machine where the user's local service (Obsidian, Postgres, whatever) is running.

Fix: replace with `host.docker.internal`. This is a special DNS name Docker Desktop provides that resolves to the host's gateway IP. Linux doesn't have this by default — Linux users need `--add-host=host.docker.internal:host-gateway` in the docker run command, or use the actual LAN IP.

Other causes to consider:

- The target service is genuinely not running. Confirm with `lsof -iTCP:<port> -sTCP:LISTEN` on the host.
- The target service binds only to `127.0.0.1` and won't accept the container's gateway IP. Some apps need to be configured to bind to `0.0.0.0` or to the docker bridge.
- Wrong port number. Default ports change between versions; check the target app's settings.

`getaddrinfo ENOTFOUND <hostname>` specifically means DNS — the hostname can't be resolved at all. Either a typo, or `host.docker.internal` isn't available (Linux without the `--add-host`).

---

## TLS / certificate errors → L4/L5 boundary

Hallmarks: `self-signed certificate`, `unable to verify the first certificate`, `x509: certificate signed by unknown authority`, `SSL handshake failed`.

What this means: the container reached the target service, started a TLS handshake, but couldn't validate the server's certificate. Very common with local-dev services that use self-signed certs (Obsidian's Local REST API ships one).

Fixes in increasing order of "ugh, but fine":

1. **Trust the cert properly** — download the cert from the target app, add it to the container's trust store. Cleanest but most work.
2. **Use the HTTP port instead of HTTPS** if the target app offers one (Obsidian Local REST API has both; HTTP is on 27123, HTTPS on 27124). On a single-user dev machine, HTTP-on-localhost is reasonable.
3. **Set the skip-verify flag** the server library uses. Common ones:
   - Node-based servers: `NODE_TLS_REJECT_UNAUTHORIZED=0` in env
   - Python `requests`-based: server may expose a `--insecure` / `VERIFY_SSL=false` flag
   - Some servers have a server-specific flag like `OBSIDIAN_INSECURE_TLS=true`

Always mention to the user that skip-verify weakens TLS — but on a single-machine local-dev setup talking to 127.0.0.1, it's a defensible tradeoff.

---

## HTTP-shape errors → L5 config

Hallmarks: `404 Not Found`, `405 Method Not Allowed`, `400 Bad Request` with a body that looks like an API contract mismatch.

What this means: connection works, auth works, but the URL path or method or body shape doesn't match what the target service expects.

Common causes:

- Wrong base URL. The server config has e.g. `OBSIDIAN_HOST=host.docker.internal` but the user is running a fork that listens on a non-default subpath.
- Wrong API version. Some MCP server images target an older version of the target app's API.
- The target app's endpoint genuinely doesn't exist (typo or a feature the user thinks exists but doesn't).

Less common, more painful:

- A reverse proxy between container and host is rewriting paths.

---

## Protocol / tool-listing errors → L2 / L3

Hallmarks: `Tool not found: foo_bar`, `Unknown tool`, `Method not found`, a tool the user *just* added doesn't appear in the list.

What this means: the gateway and client agree on the MCP protocol, but Claude is asking for a tool that isn't in the current tool list. **Tool lists are pulled at client start time**, not refreshed live.

Fix: restart Claude Desktop (or the relevant client). After restart, re-list tools and confirm the new ones are there.

If a restart doesn't bring them in, the server failed to register. Move back to Step 2 of the main workflow.

---

## Container / filesystem errors → L3

Hallmarks: `Permission denied`, `no such file or directory` for a path the user expected to exist, `bind mount failed`.

What this means: the container started but can't access something it needs. Usually a volume-mount issue — the server expects to read/write a host directory, but the directory wasn't mounted, was mounted at the wrong path, or the container's UID can't write to it.

Fix: check the server's `volumes:` block. Compare to the server's docs for what mount it expects. macOS users sometimes need to grant Docker Desktop full-disk access in System Settings → Privacy & Security.

---

## Misleading errors

The error code is usually trustworthy, but a few patterns systematically lie. Knowing them saves hours.

- **`Network is unreachable` / `ENETUNREACH` on a Dockerized MCP server**: **First suspect empty Secret**, not routing. See the Network-family section above for the two-step check. This is by far the highest-cost misleading error in this skill's domain — it sends you chasing host.docker.internal / IPv6 / extra_hosts when the actual fix is "re-paste the API key into the toolkit UI and confirm save."
- **`Connection refused` from a server that worked yesterday**: Often L5, not L4 — the target app (Obsidian, Postgres) was quit or crashed. Check `lsof -iTCP:<port> -sTCP:LISTEN` before changing the host string.
- **`401 Unauthorized` after a Docker Desktop update**: Sometimes L3 — Docker rebuilt the container and dropped your env vars because the secret reference resolved differently. Re-check the env block / Secret in the toolkit UI.
- **Server "works" but returns empty results**: Auth and network are fine, but the API token's scope doesn't permit reading what was asked. Check the token's permissions in the source app.
- **Tools intermittently disappear**: Almost always a Docker Desktop restart that didn't fully bring containers back up. `docker ps` will show the container missing.
- **Secret field shows `*****`**: This is not a reliable signal of "filled" — it can equally well be the field's placeholder. If you suspect the secret might be empty, have the user re-paste and explicitly save; don't trust the visual.

When the error doesn't fit cleanly, read it literally, look up the exact phrase in the target service's docs, and trust what the docs say over what category the error "sounds like".
