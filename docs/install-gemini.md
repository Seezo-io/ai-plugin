# Install In Gemini CLI

This repo includes `gemini-extension.json` and `GEMINI.md` as a thin Gemini CLI adapter. `GEMINI.md` references the canonical skills under `plugins/seezo/skills/`, and `gemini-extension.json` inlines the MCP server config so Gemini CLI does not need to discover skills paths.

## Public Install

After the public repository is available:

```sh
gemini extensions install https://github.com/Seezo-io/ai-plugin
```

Restart Gemini CLI after installation so extension context, skills, and MCP settings load.

## Configure MCP

The extension ships `gemini-extension.json` pointing at the intended public Seezo MCP endpoint:

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

Do not add static API-key headers. Complete the OAuth browser flow if Gemini CLI prompts for MCP authentication. For local development, temporarily replace the URL with `http://127.0.0.1:8000/mcp`.

## Verify

In Gemini CLI:

- run `/extensions list` and confirm `seezo` is installed;
- confirm the Seezo skills are available;
- confirm OAuth completes and the Seezo MCP server connects;
- ask Gemini to follow `GEMINI.md` and run the relevant Seezo workflow.
