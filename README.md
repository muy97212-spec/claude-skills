# claude-skills

My collection of [Claude Skills](https://docs.claude.com/en/docs/build-with-claude/skills) — extension bundles that give Claude (Code, Desktop, API) specialized capabilities for specific tasks.

## Skills

### [docker-mcp-debugger](docker-mcp-debugger/)

Diagnose and fix Docker-based MCP servers that aren't working — tools missing from Claude, 401/40101 auth errors, `Network is unreachable`/ENETUNREACH from the container, self-signed cert failures, Secrets that look filled in Docker MCP Toolkit's UI but aren't actually saved, and other common breakage patterns.

Built from a real debugging session where a `Network is unreachable` error turned out to be an empty Secret in disguise — that exact anti-pattern is now baked into the diagnostic flow so future-me (or future-you) doesn't spend an hour chasing the wrong layer.

**Key references inside:**
- `SKILL.md` — five-layer mental model + 6-step diagnostic workflow
- `references/error-codes.md` — error → likely layer → fix family, with the misleading-error patterns
- `references/common-pitfalls.md` — 12 ways Docker MCP setups go wrong, plus a decision tree
- `references/known-servers.md` — per-server quirks for mcp-obsidian, mcp-github, mcp-slack, mcp-postgres, etc.

## Installation

Each skill is a directory with a `SKILL.md` at its root. To install:

- **Claude Code**: copy the skill folder to `~/.claude/skills/`
- **Cowork / Claude Desktop**: drop the `.skill` file (zip with `.skill` extension) into the Skills panel
- **Claude Agent SDK**: include in your agent's skills path

## License

MIT.
