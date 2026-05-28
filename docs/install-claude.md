# Install In Claude Code

This repo is scaffolded as a Claude Code plugin. The Claude Code marketplace entry lives at `.claude-plugin/marketplace.json` and points at the canonical plugin payload under `plugins/seezo/` (which contains `.claude-plugin/plugin.json`, `skills/`, and `.mcp.json`). Claude Code auto-discovers `skills/`, `commands/`, `agents/`, `hooks/`, and `.mcp.json` from the plugin root by convention.

## Public Marketplace Install

After the public repository is available:

```sh
claude plugin marketplace add Seezo-io/ai-plugin
claude plugin install seezo@seezo
```

The first command adds the Seezo plugin marketplace. The second installs the `seezo` plugin from that marketplace.

## Local Development Install

From a local clone:

```sh
git clone https://github.com/Seezo-io/ai-plugin.git
claude --plugin-dir ./ai-plugin/plugins/seezo
```

Claude Code will load the plugin directly from the directory for development and testing.

## Configure MCP

The plugin ships `plugins/seezo/.mcp.json` pointing at the intended public Seezo MCP endpoint:

```json
{
  "mcpServers": {
    "seezo": {
      "type": "http",
      "url": "https://app.seezo.io/mcp"
    }
  }
}
```

Do not add static API-key headers. When Claude reports that the server needs authentication, use `/mcp` or the Claude Code MCP UI to complete the OAuth browser flow.

For local development against the current local MCP server:

```sh
claude mcp add --transport http --scope user --callback-port 8765 seezo-local http://127.0.0.1:8000/mcp
```

## Verify

Run:

```sh
claude plugin validate ./plugins/seezo
```

Then start Claude Code and confirm the Seezo skills appear under the plugin namespace.
