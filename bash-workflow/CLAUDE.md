# CLAUDE.md

Claude Code plugin (v1.0.0) providing bash development workflow with TDD, parallel agents, and DevOps review gates.

## Prerequisites

- bats-core (`brew install bats-core`)
- shellcheck (`brew install shellcheck`)

## Agents

| Agent | Model | Purpose |
|-------|-------|---------|
| bash-script-architect | Haiku | Production-grade bash scripting with defensive standards |
| bash-tdd-architect | inherit | TDD for shell scripts - tests before implementation |
| bash-test-runner | Haiku | Executes shellcheck + bats tests, reports failures only |
| devops-infra-lead | Sonnet | Senior DevOps review with [PASS]/[FAIL]/[NEEDS_CHANGES] verdicts |
| github-cli | Sonnet | GitHub CLI operations for git-ops |

## Commands

- `/implement <task>` - Multi-wave orchestration: explore → plan (4 perspectives) → implement → review → test → commit

## Workflow Waves

1. **Wave 1a**: Explore agent (dependency graph, feature identification)
2. **Wave 1b**: 4 Plan agents in parallel (Implementation, Testing, Security, DevOps perspectives)
3. **Wave 2**: Parallel script-architect + tdd-architect pairs (per dependency group)
4. **Wave 3**: devops-infra-lead review + bash-test-runner execution (in parallel)
5. **Wave 4**: git-ops agent (only on [PASS] verdict)

## Key Files

- `agents/bash-script-architect.md` - Defensive bash scripting standards
- `agents/bash-tdd-architect.md` - BDD test design patterns
- `agents/test-runner.md` - Test execution with failure filtering
- `agents/devops-infra-lead.md` - Review verdicts and quality gates
- `commands/implement.md` - Multi-wave orchestration logic

## Output Directory

`~/.claude/bash-workflow/`
- explorer-findings.md
- plan-implementation.md, plan-testing.md, plan-security.md, plan-devops.md
- unified-plan.md
- review.md
