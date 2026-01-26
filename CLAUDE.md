# CLAUDE.md

Claude Code plugin marketplace with seven plugins:

**Hook Plugins:**
- **security-hooks** (v1.0.1): Blocks secrets in git commits
- **todo-log** (v1.0.1): Logs TodoWrite activity to `.claude/todos.json`

**Agent/Skill Plugins:**
- **version-control** (v1.0.0, UNSTABLE): Git operations and GitHub CLI workflows
- **golang-workflow** (v1.4.0, external): Go development workflow (hosted separately)
- **bash-workflow** (v1.0.0): Bash development with TDD, parallel agents, and DevOps review

**Documentation Plugins:**
- **claude-code-guide** (v1.0.0): Claude Code documentation for plugins, hooks, MCP, skills, settings
- **slash-command-guide** (v1.0.0): Slash command creation guide with patterns and helper scripts

## CI/CD

- `validate-versions.yml` - Blocks PRs if marketplace.json versions don't match plugin.json
- `test.yml` - Runs pytest for all plugins
- `source-branch-check.yml` - Enforces staging → main PR flow
- `detect-plugin-changes.yml` → `bump-plugin-versions.yml` → `create-release-tags.yml` - Auto semantic versioning chain (requires VERSION_PAT secret)

## Quick Reference

### Commands

```bash
/plugin marketplace add /path/to/claude-plugins
/plugin install security-hooks@plugins-by-james
/plugin install todo-log@plugins-by-james
/plugin list
/hooks
claude --debug

# Set GitHub secret from current auth token (no intermediate files)
gh auth token | gh secret set SECRET_NAME
```

### Development Workflow

1. Edit plugin scripts
2. `/plugin uninstall plugin-name`
3. `/plugin install plugin-name@marketplace-name`
4. Trigger the hook event
5. Check transcript mode (Ctrl-R) or `claude --debug`

### Plugin Documentation

- [security-hooks/CLAUDE.md](security-hooks/CLAUDE.md) - Secret detection patterns, test suite, development
- [todo-log/CLAUDE.md](todo-log/CLAUDE.md) - Log format, security features, configuration
- [version-control/CLAUDE.md](version-control/CLAUDE.md) - Git agents, GitHub CLI skills
- [bash-workflow/CLAUDE.md](bash-workflow/CLAUDE.md) - TDD agents, /implement command, DevOps review
- [claude-code-guide/CLAUDE.md](claude-code-guide/CLAUDE.md) - Plugin documentation skills
- [slash-command-guide/CLAUDE.md](slash-command-guide/CLAUDE.md) - Command creation guide

## Plugin Structure

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json       # Manifest
├── hooks/
│   └── hooks.json        # Event configuration
└── scripts/
    └── script.py         # Implementation (chmod +x)
```

For complete hook system, MCP, and settings documentation, use the `/plugin` skill.

## Common Gotchas

1. **Script permissions**: Python scripts must be executable (`chmod +x`)
2. **Exit codes matter**: Exit 2 blocks in PreToolUse, other codes are errors
3. **Use CLAUDE_PROJECT_DIR**: Don't use relative paths - hooks run in different context
4. **JSON communication**: Input via stdin, structured output via stdout/stderr
5. **Hook matcher is regex**: "Bash" matches Bash tool, patterns are substring matches
6. **Timeout default**: Hooks timeout after 30 seconds unless configured
7. **Reinstall after changes**: Plugin code isn't hot-reloaded
