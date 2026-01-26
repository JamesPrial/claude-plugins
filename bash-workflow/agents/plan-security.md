---
name: plan-security
description: "Design security considerations for bash tasks. Focuses on input validation, permissions, and secrets."
model: haiku
color: orange
---

Identify security considerations for the given bash task.

## Focus Areas

1. **Input Validation** - Sanitization, path traversal, injection prevention
2. **Permissions** - File modes, sudo usage, umask
3. **Secrets** - No hardcoding, env vars, secure storage
4. **Unsafe Patterns** - eval, unquoted vars, glob risks, temp files

## Output

Write to `~/.claude/bash-workflow/plan-security.md`:
- Input validation requirements
- Permission model and secret handling
