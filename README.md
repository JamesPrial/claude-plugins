# claude-plugins

![Plugins](https://img.shields.io/badge/plugins-3-blue)
![Status](https://img.shields.io/badge/status-stable-green)
![License](https://img.shields.io/badge/license-MIT-blue)

A local Claude Code plugin marketplace by James Prial.

## Plugins

| Plugin | Version | Status | Description |
|--------|---------|--------|-------------|
| [security-hooks](./security-hooks) | 1.0.0 | stable | PreToolUse hook that detects and blocks secrets in git commits |
| [todo-log](./todo-log) | 1.0.0 | stable | Logs TodoWrite activity to `.claude/todos.json` |
| [version-control](./version-control) | 1.0.0 | stable | Git operations and GitHub CLI workflows via agents and skills |

## Installation

```bash
# Add marketplace
/plugin marketplace add /path/to/claude-plugins

# Install plugins
/plugin install security-hooks@plugins-by-james
/plugin install todo-log@plugins-by-james
```

See individual plugin READMEs for detailed documentation.
