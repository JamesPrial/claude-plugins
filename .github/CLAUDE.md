# .github/CLAUDE.md

GitHub Actions workflows for the plugin marketplace.

## Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `validate-versions.yml` | PR/push when marketplace.json or plugin.json changes | Blocks PRs if versions mismatch |
| `test.yml` | PR to main, push to main/staging | Runs pytest for all plugins |
| `source-branch-check.yml` | PR to main | Enforces staging → main flow |

## Version Validation

The `validate-versions.yml` workflow uses a matrix strategy to check each plugin in parallel:
- **Local plugins**: Compares marketplace.json version to `{source}/.claude-plugin/plugin.json`
- **External plugins**: Shallow clones the URL and compares versions

## Adding New Workflows

1. Create `.github/workflows/<name>.yml`
2. Use `ubuntu-latest`, `actions/checkout@v4`
3. Set explicit `permissions` (principle of least privilege)
4. Add `timeout-minutes` to prevent runaway jobs
