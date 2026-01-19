# Contributing to claude-plugins

Thanks for your interest in contributing! This guide covers how to add new plugins or improve existing ones.

## Plugin Structure

Every plugin must follow this structure:

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json       # Required: plugin manifest
├── hooks/
│   └── hooks.json        # Hook event configuration
├── scripts/
│   └── script.py         # Implementation (must be executable)
├── README.md             # User documentation
└── CLAUDE.md             # Developer documentation
```

### plugin.json

```json
{
  "name": "plugin-name",
  "version": "1.0.0",
  "description": "Brief description of what the plugin does",
  "author": "Your Name",
  "hooks": "hooks/hooks.json"
}
```

### hooks.json

```json
{
  "description": "Brief description of what the hooks do",
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "ToolName",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/scripts/script.py",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

Note: Use `PreToolUse` for hooks that run before a tool (can block), or `PostToolUse` for hooks that run after.

## Development Setup

1. Clone the repository
2. Add the local marketplace:
   ```bash
   /plugin marketplace add /path/to/claude-plugins
   ```
3. Install your plugin:
   ```bash
   /plugin install your-plugin@plugins-by-james
   ```

## Testing

All plugins must have tests. Run tests with pytest:

```bash
python -m pytest plugin-name/scripts/test_*.py -v
```

### Testing Hooks Manually

```bash
echo '{"tool_name": "Bash", "tool_input": {"command": "example"}}' | ./scripts/your_script.py
```

## Code Guidelines

### Python Scripts

- Must be executable (`chmod +x`)
- Read input from stdin as JSON
- Output structured JSON to stdout/stderr
- Use appropriate exit codes:
  - `0`: Success (allow or neutral)
  - `2`: Block the action (PreToolUse) or error

### Hook Best Practices

- Use `${CLAUDE_PLUGIN_ROOT}` or `${CLAUDE_PROJECT_DIR}` for paths
- Never use relative paths (hooks run in different contexts)
- Set reasonable timeouts (default is 30 seconds)
- Fail closed on errors (exit 2)

## Pull Request Process

1. Create a branch for your changes
2. Ensure all tests pass
3. Update documentation (README.md, CLAUDE.md)
4. Update version in plugin.json if modifying existing plugins
5. Submit a PR with a clear description of changes

## Adding a New Plugin

1. Create the plugin directory structure
2. Implement the hook script with tests
3. Add README.md with installation and usage docs
4. Add CLAUDE.md with developer documentation
5. Update the main README.md plugin table
6. Submit a PR

## Questions?

Open an issue for questions or to discuss new plugin ideas before implementation.
