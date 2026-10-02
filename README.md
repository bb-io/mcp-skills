# Blackbird plugin

Blackbird is the AI orchestration, context, and governance layer around multilingual content. This package contains its shared skills for Claude Code, Codex, and ChatGPT.

## Layout

- `skills/` is the canonical, provider-neutral workflow content.
- `.claude-plugin/plugin.json` enables Claude Code plugin packaging.
- `.codex-plugin/plugin.json` is the Codex compatibility manifest.
- `plugin.json` is the portable OpenAI plugin manifest.
- `.mcp.json` and `mcp.json` are intentionally separate compatibility files. Keep their MCP server definitions in sync when configuring a production endpoint.

## Configure the MCP server

Both MCP configuration files target the staging endpoint at `https://staging.mcp.blackbird.io/mcp`. The portable `mcp.json` uses the Agent Plugins `streamable-http` transport required by OpenAI plugin packaging; `.mcp.json` retains the compatibility format for Claude Code and Codex. Use a separate production package for `https://mcp.blackbird.io/mcp`, and do not add API keys or other secrets to this package.

The included skill assumes that the MCP connection is already available to the model.
