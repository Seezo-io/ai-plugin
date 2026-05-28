# Install In Cursor

This repo includes `.cursor-plugin/plugin.json` and `.cursor-plugin/marketplace.json` as a thin Cursor adapter. The Cursor plugin manifest points at the canonical payload under `plugins/seezo/` (skills at `./plugins/seezo/skills/`, MCP at `./plugins/seezo/.mcp.json`). Cursor's own MCP discovery files are `.cursor/mcp.json` and `mcp.json` at the repo root.

## Plugin Install

After the public repository is available, install Seezo from the Cursor Marketplace when the listing is published.

For direct repo installation, add `https://github.com/Seezo-io/ai-plugin` as a plugin marketplace source in Cursor Settings > Plugins if your Cursor build supports external plugin marketplaces. For local testing, add this repository as a local plugin marketplace or local plugin path.

## MCP Setup

For project-specific MCP setup, use the checked-in `.cursor/mcp.json`:

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

For global MCP setup, copy the same body into `~/.cursor/mcp.json`.

Do not add static API-key headers. Cursor should start the OAuth browser flow when the server challenges the first connection. For local development, temporarily replace the URL with `http://127.0.0.1:8000/mcp`.

## Verify

In Cursor:

- open MCP settings and confirm the `seezo` server is listed;
- complete OAuth if Cursor prompts for authentication;
- confirm tools/resources/prompts appear after the server connects;
- confirm the Seezo skills are available from the plugin;
- ask Cursor to use the relevant Seezo workflow for a security task.
