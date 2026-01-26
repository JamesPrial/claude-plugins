# claude-code-guide

Claude Code plugin providing comprehensive documentation for Claude Code extensibility.

## Features

- **Plugin Development** - Create plugins with commands, agents, hooks, skills, and MCP servers
- **Hook System** - Event-driven automation with PreToolUse, PostToolUse, and more
- **MCP Integration** - Connect external tools via Model Context Protocol
- **Skills Reference** - Agent skill development patterns
- **Settings Guide** - Configuration options and permission management

## Installation

```bash
/plugin install claude-code-guide@plugins-by-james
```

## Usage

The skill is automatically invoked when you discuss plugins, hooks, MCP, or Claude Code configuration. Example triggers:

- "How do I create a plugin hook?"
- "What's the plugin.json schema?"
- "How do I add an MCP server?"
- "Configure Claude Code settings"

## Skills Included

| Skill | Description |
|-------|-------------|
| `claude-code-plugins` | 9 reference docs covering the complete plugin ecosystem |

## Reference Documentation

- [Plugins](skills/claude-code-plugins/references/plugins.md) - Getting started tutorials
- [Plugin Reference](skills/claude-code-plugins/references/plugins-reference.md) - Technical specifications
- [Hooks](skills/claude-code-plugins/references/hooks.md) - Event handling automation
- [MCP](skills/claude-code-plugins/references/mcp.md) - External tool integration
- [Settings](skills/claude-code-plugins/references/settings.md) - Configuration options

## License

MIT
