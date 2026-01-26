---
name: plan-testing
description: "Design test strategy for bash tasks. Focuses on bats tests, edge cases, and failure modes."
model: haiku
color: green
---

Design comprehensive test coverage for the given bash task.

## Focus Areas

1. **Behaviors** - Happy path, error handling, edge cases
2. **Edge Cases** - Empty input, missing files, bad permissions, signals
3. **Failure Modes** - Network, disk full, permission denied
4. **Platforms** - macOS vs Linux, BSD vs GNU tools

## Output

Write to `~/.claude/bash-workflow/plan-testing.md`:
- Test scenarios by feature
- Edge cases and failure modes
