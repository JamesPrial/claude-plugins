# bash-workflow

Claude Code plugin providing bash development workflow with TDD, parallel agent orchestration, and DevOps review gates.

## Features

- **Test-Driven Development** - Write tests before implementation
- **Parallel Execution** - Multiple agents work independently when safe
- **DevOps Review Gate** - Production-grade quality checks
- **Multi-wave Orchestration** - Explore → Plan → Implement → Review → Test → Commit

## Installation

```bash
/plugin install bash-workflow@plugins-by-james
```

## Prerequisites

- **bats-core** - Bash testing framework
  ```bash
  brew install bats-core  # macOS
  apt install bats        # Debian/Ubuntu
  ```
- **shellcheck** - Shell script static analysis
  ```bash
  brew install shellcheck  # macOS
  apt install shellcheck   # Debian/Ubuntu
  ```

## Usage

### /implement Command

Orchestrates the full development workflow:

```
/implement Create a backup script with rotation and compression
```

Workflow waves:
1. **Wave 1a**: Explore codebase dependencies
2. **Wave 1b**: 4 parallel planners (Implementation, Testing, Security, DevOps)
3. **Wave 2**: Parallel script-architect + tdd-architect pairs
4. **Wave 3**: DevOps review + test runner
5. **Wave 4**: Git operations (only on [PASS])

## Agents Included

| Agent | Model | Purpose |
|-------|-------|---------|
| bash-script-architect | Haiku | Production-grade bash scripting |
| bash-tdd-architect | Inherit | TDD for shell scripts |
| bash-test-runner | Haiku | Executes shellcheck + bats tests |
| devops-infra-lead | Sonnet | Senior DevOps review |
| github-cli | Sonnet | GitHub CLI operations |

## Scripting Standards

Scripts produced by bash-script-architect follow these standards:

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'

# Mandatory safety: quoted variables, error handling, cleanup traps
```

## License

MIT
