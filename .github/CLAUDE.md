# .github/CLAUDE.md

GitHub Actions workflows for the plugin marketplace.

## Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `validate-versions.yml` | PR/push when marketplace.json or plugin.json changes | Blocks PRs if versions mismatch |
| `test.yml` | PR to main, push to main/staging | Runs pytest for all plugins |
| `source-branch-check.yml` | PR to main | Enforces staging → main flow |
| `detect-plugin-changes.yml` | Push to main (plugin paths) | Detects changes, dispatches version bump |
| `bump-plugin-versions.yml` | repository_dispatch: bump-versions | Bumps versions, commits, dispatches tagging |
| `create-release-tags.yml` | repository_dispatch: create-tags | Creates and pushes git tags |

## Version Validation

The `validate-versions.yml` workflow uses a matrix strategy to check each plugin in parallel:
- **Local plugins**: Compares marketplace.json version to `{source}/.claude-plugin/plugin.json`
- **External plugins**: Shallow clones the URL and compares versions

## Auto Semantic Versioning

Three workflows chain together via `repository_dispatch` to automatically version plugins:

```
Push to main (plugin changes)
         │
         ▼
┌─────────────────────────┐
│ detect-plugin-changes   │  Detects which plugins changed
│ Analyzes PR labels or   │  Determines bump type (major/minor/patch)
│ commit messages         │
└───────────┬─────────────┘
            │ repository_dispatch: bump-versions
            ▼
┌─────────────────────────┐
│ bump-plugin-versions    │  Calculates new versions
│ Updates plugin.json &   │  Creates chore commit
│ marketplace.json        │
└───────────┬─────────────┘
            │ repository_dispatch: create-tags
            ▼
┌─────────────────────────┐
│ create-release-tags     │  Creates git tags (plugin@vX.Y.Z)
│ Pushes tags to origin   │
└─────────────────────────┘
```

### Bump Type Detection

Priority order:
1. **PR labels**: `major`/`breaking` → major, `feat`/`feature`/`minor` → minor, `fix`/`bugfix`/`patch` → patch
2. **Commit message**: `BREAKING CHANGE` → major, `feat:` → minor
3. **Default**: patch

### Prerequisites

Requires `VERSION_PAT` repository secret with:
- Contents: Read and write
- Pull requests: Read
- Workflows: Read and write

### Manual Bumps

If you manually update a `plugin.json` version before merging, the workflow detects the mismatch and syncs `marketplace.json` without auto-incrementing.

### External Plugins

External plugins (like `golang-workflow`) are skipped - they're versioned in their own repositories.

## Adding New Workflows

1. Create `.github/workflows/<name>.yml`
2. Use `ubuntu-latest`, `actions/checkout@v4`
3. Set explicit `permissions` (principle of least privilege)
4. Add `timeout-minutes` to prevent runaway jobs
