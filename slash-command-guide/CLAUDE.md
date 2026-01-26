# CLAUDE.md

Claude Code plugin (v1.0.0) for creating custom slash commands.

## Skills

- **claude-code-slash-commands** - Command syntax, argument handling, patterns, and helper script

## Skill Triggers

The skill is auto-invoked when relevant. Example triggers:
- "Create a slash command for..."
- "How do I use $ARGUMENTS?"
- "What's the allowed-tools syntax?"
- "Make a /deploy command"

## Key Files

| File | Purpose |
|------|---------|
| SKILL.md | Main syntax reference with examples |
| references/patterns.md | Ready-to-use command templates |
| references/skills-vs-commands.md | When to use skills vs commands |
| scripts/init_command.py | Scaffold new commands |

## Helper Script Usage

```bash
# Create project-level command
python3 scripts/init_command.py my-command --project

# Create user-level command with tools
python3 scripts/init_command.py deploy --user --with-tools
```
