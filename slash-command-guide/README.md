# slash-command-guide

Claude Code plugin for creating custom slash commands.

## Features

- **Command Syntax** - YAML frontmatter options and prompt body structure
- **Argument Handling** - `$ARGUMENTS`, `$1`, `$2` for positional args
- **Bash Execution** - Inline bash with `!` prefix
- **File References** - Include file contents with `@` prefix
- **Ready-to-use Patterns** - Copy-paste templates for common commands
- **Helper Script** - `init_command.py` for scaffolding new commands

## Installation

```bash
/plugin install slash-command-guide@plugins-by-james
```

## Usage

The skill is automatically invoked when you want to create slash commands. Example triggers:

- "Create a slash command for..."
- "How do I use $ARGUMENTS?"
- "What frontmatter options are available?"
- "Make a /commit command"

## Skills Included

| Skill | Description |
|-------|-------------|
| `claude-code-slash-commands` | Command syntax, patterns, and helper script |

## Quick Start

Create a command in `.claude/commands/` (project) or `~/.claude/commands/` (personal):

```markdown
---
allowed-tools: Bash(git:*)
argument-hint: [message]
description: Commit with message
---

Create a commit with message: $ARGUMENTS
```

## Helper Script

Scaffold a new command:

```bash
python3 scripts/init_command.py review --project --with-tools
```

## License

MIT
