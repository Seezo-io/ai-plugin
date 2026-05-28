# Install In GitHub Copilot / VS Code

GitHub Copilot in VS Code uses VS Code MCP configuration and Agent Skills discovery. This repo ships both `.vscode/mcp.json` and `.vscode/settings.json`.

## Workspace MCP Setup

Use the checked-in `.vscode/mcp.json` in this workspace:

```json
{
  "servers": {
    "seezo": {
      "type": "http",
      "url": "https://app.seezo.io/mcp"
    }
  }
}
```

Do not add static API-key headers. VS Code supports OAuth for remote MCP servers; complete the browser flow if VS Code prompts for authentication. If a future VS Code build requires client-specific OAuth metadata, follow the then-current VS Code MCP documentation instead of adding secrets to this repo.

## Agent Skills Setup

The canonical `plugins/seezo/skills/` directory is enabled for VS Code through `.vscode/settings.json`:

```json
{
  "chat.agentSkillsLocations": {
    "plugins/seezo/skills": true,
    ".github/skills": true,
    ".claude/skills": true,
    ".agents/skills": true,
    "~/.copilot/skills": true,
    "~/.claude/skills": true,
    "~/.agents/skills": true
  }
}
```

This keeps `plugins/seezo/skills/start-assessment/SKILL.md` and `plugins/seezo/skills/validate-implementation/SKILL.md` as the single shared skill source for Copilot and the plugin clients.

## Verify

In VS Code:

- enable GitHub Copilot Chat Agent mode;
- run `/skills` and confirm the Seezo skills are listed;
- confirm the `seezo` MCP server is enabled in the MCP server list;
- complete OAuth if VS Code opens an authorization browser window;
- confirm Seezo tools appear in Copilot's available tools after the server connects.
