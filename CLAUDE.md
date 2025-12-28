# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Claude Code plugin marketplace** containing two production-ready plugins that extend Claude Code functionality through event-driven hooks:

- **security-hooks** (v1.0.0): Security plugin that detects and blocks potential secrets in git commits
- **todo-log** (v0.2.0): Productivity plugin that automatically logs TodoWrite tool usage to `.claude/todos.json`

## Development Commands

### Plugin Management

```bash
# Add this local marketplace (use your actual path)
/plugin marketplace add /path/to/claude-plugins

# Install a plugin from marketplace
/plugin install security-hooks@plugins-by-james
/plugin install todo-log@plugins-by-james

# List installed plugins
/plugin list

# Validate hooks are registered
/hooks

# Debug mode to see hook execution details
claude --debug
```

### Testing Workflow

1. Make changes to plugin scripts or configuration
2. Uninstall plugin: `/plugin uninstall plugin-name`
3. Reinstall plugin: `/plugin install plugin-name@marketplace-name`
4. Test manually by triggering the hook event
5. Check debug output or transcript mode (Ctrl-R) for hook feedback

See each plugin's CLAUDE.md for test commands.

## Architecture

### Plugin Structure

Each plugin follows this structure:

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json       # Manifest with name, version, description
├── hooks/
│   └── hooks.json        # Hook event configuration
└── scripts/
    └── script.py         # Python implementation (chmod +x)
```

### Hook System

Hooks are **event-driven handlers** that respond to Claude Code events:

**Event Types:**
- `PreToolUse`: Runs before a tool executes (can block execution)
- `PostToolUse`: Runs after a tool completes (for logging/side effects)

**Hook Configuration (hooks.json):**
```json
[
  {
    "event": "PreToolUse" | "PostToolUse",
    "matchers": [
      {
        "matcher": "Bash",  // Tool name to match
        "type": "command",
        "command": "${CLAUDE_PLUGIN_ROOT}/scripts/script.py",
        "timeout": 30
      }
    ]
  }
]
```

**Hook Input (stdin):**
Hooks receive JSON with complete session context including tool parameters, conversation history, and environment.

**Hook Output:**
- **Exit Code 0**: Success, continue normally
- **Exit Code 2**: Block tool execution and show stderr to Claude (PreToolUse only)
- **Other exit codes**: Error state
- **stdout/stderr**: Displayed to user in transcript mode or debug mode

**Environment Variables:**
- `CLAUDE_PROJECT_DIR`: Project root directory (use for file operations)
- `CLAUDE_PLUGIN_ROOT`: Plugin installation directory (use for accessing plugin files)
- `TODO_LOG_PATH`: Custom log file path for todo-log plugin (default: `.claude/todos.json`)

### Plugins

**security-hooks** (v1.0.0): PreToolUse hook that scans staged files for 24+ secret patterns before git commits. Blocks commits containing detected secrets.
→ See [security-hooks/CLAUDE.md](security-hooks/CLAUDE.md) for patterns, test suite, and development details.

**todo-log** (v0.2.0): PostToolUse hook that logs TodoWrite tool usage to `.claude/todos.json` with timestamps and session metadata.
→ See [todo-log/CLAUDE.md](todo-log/CLAUDE.md) for log format, security features, and configuration.

## Technology Stack

- **Language:** Python 3 (stdlib only, no external dependencies)
- **Configuration:** JSON for all manifests, hooks, and data
- **Scripts:** Must have execute permissions (`chmod +x`)

## Key Development Patterns

### Zero-Dependency Implementation

All Python scripts use **only standard library**. This ensures:
- No installation/setup required beyond Python 3
- No dependency management or version conflicts
- Plugins work immediately after installation

### Fail-Safe Hook Design

**Exit code conventions prevent unintended side effects:**
- PreToolUse hooks must explicitly return exit code 2 to block
- PostToolUse hooks should never block (exit 0 or error)
- Stderr messages guide Claude on what went wrong

**Example from security-hooks:**
```python
if secrets_found:
    print(json.dumps({"errors": [...]}), file=sys.stderr)
    sys.exit(2)  # Block commit
else:
    print(json.dumps({"message": "No secrets detected"}))
    sys.exit(0)  # Allow commit
```

### Marketplace Distribution

**Marketplace manifest (.claude-plugin/marketplace.json):**
```json
{
  "name": "plugins-by-james",
  "owner": { "name": "James Prial" },
  "plugins": [
    {
      "name": "security-hooks",
      "source": "./security-hooks",
      "description": "PreToolUse hook that detects and blocks secrets in git commits",
      "version": "1.0.0"
    }
  ]
}
```

Plugins are distributed via local marketplace, allowing versioning and centralized management.

## Plugin Skill Reference

The `.claude/skills/plugin/` directory contains **comprehensive reference documentation** (5,570+ lines):

- `hooks.md` (1093 lines): Complete hook system documentation
- `mcp.md` (1296 lines): MCP server integration
- `marketplaces.md` (432 lines): Marketplace creation and management
- `plugins.md` (391 lines): Plugin development tutorials
- `settings.md` (406 lines): Configuration options
- `skills.md` (607 lines): Agent skills system
- `subagents.md` (479 lines): Subagent details

**To access:** Use the `/plugin` skill when working on plugin development.

## Common Gotchas

1. **Script permissions**: Python scripts must be executable (`chmod +x`)
2. **Exit codes matter**: Exit 2 blocks in PreToolUse, other codes are errors
3. **Use CLAUDE_PROJECT_DIR**: Don't use relative paths or cwd - hooks run in different context
4. **JSON communication**: Input via stdin, structured output via stdout/stderr
5. **Hook matcher is regex**: "Bash" matches Bash tool, patterns like "git commit" are substring matches
6. **Timeout default**: Hooks timeout after 30 seconds unless configured otherwise
7. **Reinstall after changes**: Plugin code isn't hot-reloaded - must reinstall to test changes
