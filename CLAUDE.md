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

**Running the security-hooks test suite:**
```bash
# Run with pytest
python -m pytest security-hooks/scripts/test_check_secrets.py -v

# Or run directly (includes unittest runner)
python security-hooks/scripts/test_check_secrets.py
```

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

### security-hooks Architecture

**Type:** PreToolUse hook on Bash tool matching "git commit" commands

**How it works:**
1. Parses `.env` files to extract secret values
2. Gets list of staged files via git (uses staging area, not disk)
3. Scans staged files for:
   - Pattern-based secrets (18+ patterns)
   - Hardcoded values from `.env` files
4. Exits with code 2 if secrets found (blocks commit)

**Pattern Detection (18+ patterns):**
- AWS Access Key ID: `AKIA[0-9A-Z]{16}`
- AWS Secret Access Key: 40-char base64
- OpenAI: `sk-[a-zA-Z0-9]{48}`
- Anthropic: `sk-ant-[a-zA-Z0-9-_]{90,}`
- GitHub: `ghp_`, `gho_`, `ghu_`, `ghs_`, `ghr_` prefixes
- Slack: `xox[baprs]-` prefixes
- Stripe: `sk_live_`, `rk_live_`
- Google API: `AIza[0-9A-Za-z-_]{35}`
- SendGrid: `SG\.[a-zA-Z0-9-_]{22,}`
- Discord, npm, PyPI, Twilio, Mailgun tokens
- Bearer tokens, private keys (RSA, DSA, EC, OpenSSH)
- Database connection strings (PostgreSQL, MySQL, MongoDB)

**Filtering Logic:**
- Skips binary files (30+ extensions) and `.env` files themselves
- Skips files >10MB
- Filters out short values (<8 chars) and common non-secrets
- Reports line numbers for easy remediation

### todo-log Architecture

**Type:** PostToolUse hook on TodoWrite tool

**How it works:**
1. Receives TodoWrite tool result via stdin
2. Extracts todo list from tool result
3. Adds metadata: timestamp (ISO 8601), session_id, cwd
4. Appends to log file as JSON array
5. Creates parent directories if needed

**Configuration:**
- Default log path: `.claude/todos.json`
- Custom path via `TODO_LOG_PATH` environment variable

**Output Format:**
```json
[
  {
    "timestamp": "2025-11-14T10:30:45.123Z",
    "session_id": "abc123def456",
    "cwd": "/home/user/project",
    "todos": [
      {
        "content": "Task description",
        "status": "pending" | "in_progress" | "completed",
        "activeForm": "Active form of task"
      }
    ]
  }
]
```

**Error Handling:**
Gracefully handles corrupted JSON files by initializing as empty array.

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
