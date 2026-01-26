# claude-plugins

![Plugins](https://img.shields.io/badge/plugins-7-blue)
![Status](https://img.shields.io/badge/status-stable-green)
![License](https://img.shields.io/badge/license-MIT-blue)

A local Claude Code plugin marketplace by James Prial.

## Plugins

| Plugin | Version | Status | Description |
|--------|---------|--------|-------------|
| [security-hooks](./security-hooks) | 1.0.1 | stable | PreToolUse hook that detects and blocks secrets in git commits |
| [todo-log](./todo-log) | 1.0.1 | stable | Logs TodoWrite activity to `.claude/todos.json` |
| [version-control](./version-control) | 1.0.0 | UNSTABLE | Git operations and GitHub CLI workflows via agents and skills |
| [golang-workflow](https://github.com/JamesPrial/golang-plugin) | 1.4.0 | stable | Go development workflow with specialized agents (external) |
| [bash-workflow](./bash-workflow) | 1.0.0 | stable | Bash development with TDD, parallel agents, and DevOps review |
| [claude-code-guide](./claude-code-guide) | 1.0.0 | stable | Claude Code documentation for plugins, hooks, MCP, skills, settings |
| [slash-command-guide](./slash-command-guide) | 1.0.0 | stable | Slash command creation guide with patterns and helper scripts |

## Installation

```bash
# Add marketplace
/plugin marketplace add /path/to/claude-plugins

# Install plugins
/plugin install security-hooks@plugins-by-james
/plugin install todo-log@plugins-by-james
```

See individual plugin READMEs for detailed documentation.
