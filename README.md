# claude-plugins

A local Claude Code plugin marketplace by James Prial.

## Plugins

| Plugin | Version | Status | Description |
|--------|---------|--------|-------------|
| [security-hooks](./security-hooks) | 1.0.0 | stable | PreToolUse hook that detects and blocks secrets in git commits |
| [todo-log](./todo-log) | 0.2.0 | unstable | Logs TodoWrite activity to `.claude/todos.json` |

## Installation

```bash
# Add marketplace
/plugin marketplace add /path/to/claude-plugins

# Install plugins
/plugin install security-hooks@plugins-by-james
/plugin install todo-log@plugins-by-james
```

See individual plugin READMEs for detailed documentation.
