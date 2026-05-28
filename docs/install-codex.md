# Install In Codex

This repo is scaffolded as a Codex plugin. The Codex marketplace entry lives at `.agents/plugins/marketplace.json` and points at the canonical plugin payload under `plugins/seezo/` (which contains `.codex-plugin/plugin.json`, `skills/`, and `.mcp.json`).

## Public Install

After the public repository is available:

```sh
codex plugin marketplace add Seezo-io/ai-plugin
codex
```

Inside Codex, open `/plugins`, select `Seezo`, and install it.

## Local Development Install

From a local clone:

```sh
git clone https://github.com/Seezo-io/ai-plugin.git
codex plugin marketplace add ./ai-plugin
codex
```

Then use `/plugins` to install or enable `Seezo` from the local marketplace.

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

Do not add static API-key headers. When the MCP server is live, authenticate with OAuth through Codex:

```sh
codex mcp login seezo
```

For local development against the current local MCP server, use a temporary Codex home:

```sh
mkdir -p /private/tmp/seezo-codex-mcp-test
CODEX_HOME=/private/tmp/seezo-codex-mcp-test codex mcp add seezo-local --url http://127.0.0.1:8000/mcp
CODEX_HOME=/private/tmp/seezo-codex-mcp-test codex mcp login seezo-local
```

## Verify

In Codex:

- confirm the `Seezo` plugin is enabled;
- ask Seezo to use the relevant workflow for a security task;
- confirm the relevant skill is available;
- confirm OAuth completes and Seezo MCP tools are listed.
