# Seezo AI Plugin [beta]

Official Seezo plugin for AI clients. Connects shared agent skills to the remote Seezo MCP so Seezo security workflows are available from Claude Code, Codex, Cursor, GitHub Copilot / VS Code, and Gemini CLI.

> Status: **beta**. APIs, skill names, and manifests may still change. Pin a tag in CI if you need stability.

## Install

| Client | Install guide |
| --- | --- |
| Claude Code | [docs/install-claude.md](docs/install-claude.md) |
| Codex | [docs/install-codex.md](docs/install-codex.md) |
| Cursor | [docs/install-cursor.md](docs/install-cursor.md) |
| GitHub Copilot / VS Code | [docs/install-copilot.md](docs/install-copilot.md) |
| Gemini CLI | [docs/install-gemini.md](docs/install-gemini.md) |

## Skills

- `start-assessment` — start a Seezo assessment from a plan, spec, RFC, ADR, or design doc before implementation begins.
- `validate-implementation` — validate implemented code against a prior Seezo assessment before declaring work done.

The skills are host-agnostic. They keep outputs evidence-led, avoid leaking customer data or proprietary rules, and use the Seezo MCP only when it is configured, authenticated, and relevant.

## MCP

The committed MCP configs target the public Seezo endpoint:

```
https://app.seezo.io/mcp
```

## Layout

The canonical plugin payload is a single source of truth under `plugins/seezo/`. Client-specific files at the repo root are thin adapters around it.

```
plugins/seezo/
  .claude-plugin/plugin.json     # Claude Code plugin manifest
  .codex-plugin/plugin.json      # Codex plugin manifest
  .mcp.json                      # MCP server config (Claude Code + Codex)
  skills/
    start-assessment/SKILL.md
    validate-implementation/SKILL.md
```

Adapters at the root:

| Path | Used by |
| --- | --- |
| `.claude-plugin/marketplace.json` | Claude Code marketplace entry → `plugins/seezo/` |
| `.agents/plugins/marketplace.json` | Codex marketplace entry → `plugins/seezo/` |
| `.cursor-plugin/plugin.json`, `.cursor-plugin/marketplace.json` | Cursor |
| `.cursor/mcp.json`, `mcp.json` | Cursor MCP discovery |
| `.vscode/mcp.json`, `.vscode/settings.json` | GitHub Copilot / VS Code (MCP + skill location) |
| `gemini-extension.json`, `GEMINI.md` | Gemini CLI |

## License

MIT — see [LICENSE](LICENSE).
