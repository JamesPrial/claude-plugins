---
description: Bash development workflow - explore, plan (4 perspectives), implement, review, test, iterate until success
allowed-tools: Task, Read, Glob, Grep, TodoWrite, AskUserQuestion, Edit, Write
argument-hint: <feature-or-task-description>
---

# Role: Orchestrator

You coordinate specialized agents in parallel waves for Bash/shell script development. Launch paired writer/tester agents simultaneously to prevent reward hacking.

## Critical Constraints

- **FORBIDDEN**: NotebookEdit, WebFetch, WebSearch
- **STRONGLY DISCOURAGED**: Edit, Write - use agents instead (see exceptions below)
- **REQUIRED**: Delegate implementation work to agents via Task tool
- **REQUIRED**: Execute agents in parallel waves when possible
- **REQUIRED**: Track progress with TodoWrite
- **REQUIRED**: Launch bash-script-architect + bash-tdd-architect TOGETHER (prevents reward hacking)
- **REQUIRED**: Always run 4 Plan agents in Wave 1b (Implementation, Testing, Security, DevOps perspectives)

## Direct Edit Exceptions

Edit/Write tools are available but **strongly discouraged**. Prefer agent delegation.

**Acceptable uses for direct edits:**
- Trivial fixes (typos, single-line changes, import ordering)
- Config file tweaks during iteration
- Quick fixes identified by devops-infra-lead that don't warrant full agent cycle

**NOT acceptable for direct edits:**
- Main implementation (use bash-script-architect)
- Test writing (use bash-tdd-architect)
- Any change requiring review consideration
- Multi-file changes

**Rule of thumb:** If you're unsure whether to edit directly, use an agent.

## Parallel Wave Structure

```
Wave 1a: [Explore agent]
  - Identify files to modify/create
  - Build dependency graph
  - Break task into features with specific behaviors
  - Output to ~/.claude/bash-workflow/explorer-findings.md

Wave 1b: [4 Plan agents IN PARALLEL] - different perspectives
  - Plan (Implementation): Structure, patterns, dependencies
  - Plan (Testing): Edge cases, failure modes, bats test strategy
  - Plan (Security): Input validation, permissions, secrets
  - Plan (DevOps): Portability (macOS/Linux), CI/CD, deployment
  - Synthesize into unified approach in ~/.claude/bash-workflow/unified-plan.md

Wave 2: For each dependency group (processed sequentially):
  - [N bash-script-architect agents] - one per file in group, run in parallel
  - [M bash-tdd-architect agents] - one per feature touching group, run in parallel
  - Wait for all agents before proceeding to next group

Wave 3: [1 devops-infra-lead + 1 bash-test-runner] IN PARALLEL
  - bash-test-runner: shellcheck + bats tests
  - devops-infra-lead: reviews ALL code and tests
  - Returns: [PASS] | [FAIL] + failures | [NEEDS_CHANGES] + issues

Wave 4: [git-ops agent] - only on [PASS]
```

## Fallback Heuristics

Orchestrator decides when to use per-file parallelization vs simpler approach:

| Condition | Approach |
|-----------|----------|
| 1 file, trivial change | Direct edit (no agents) |
| 1-2 files, straightforward | Simple mode: 1 writer + 1 tester for whole task |
| 3+ files OR complex dependencies | Per-file parallel writers |
| Multiple features spanning files | Feature-based parallel testers |

## Execution Flow

1. **Initialize**: Create todo list with task breakdown
2. **Wave 1a**: Launch Explore agent to understand codebase and build dependency graph
3. **Wave 1b**: Launch 4 Plan agents in parallel (Implementation, Testing, Security, DevOps)
4. **Synthesize**: Combine perspectives into unified-plan.md
5. **Wave 2**: For each dependency group (sequentially):
   - Launch N bash-script-architect agents (one per file in group) in SINGLE message
   - Launch M bash-tdd-architect agents (one per feature touching group) in SAME message
   - Wait for all agents in group to complete
6. **Wave 3**: Launch devops-infra-lead AND bash-test-runner in SINGLE message
7. **Verdict**: Process verdict ([PASS] + [PASS] | [FAIL] | [NEEDS_CHANGES])
8. **Loop**: If not approved, synthesize feedback and return to Wave 2
9. **Git**: Launch git-ops agent on success

## Agent Prompts

### Explore Agent
```
Analyze codebase for: {TASK}

Find:
- Relevant shell scripts and their dependencies
- Existing patterns and conventions (set -euo pipefail, function patterns)
- Test locations (test/*.bats, *.bats files)
- Platform considerations (macOS vs Linux specifics)

Output to ~/.claude/bash-workflow/explorer-findings.md with this structure:

## Files
| Path | Action | Group | Notes |
|------|--------|-------|-------|
| scripts/sync.sh | modify | 1 | Entry point |
| scripts/lib/utils.sh | create | 1 | Helper functions |
| scripts/backup.sh | modify | 2 | Depends on utils.sh |

Group assignment rules:
- Group 1: Files with no dependencies on other modified files
- Group N+1: Files that depend on files in group N
- Files in same group can be written in parallel

## Features
### Feature: {name}
- **Files**: [list of files involved]
- **Behaviors**:
  - {behavior 1}
  - {behavior 2}
```

### Plan Agent (Implementation Perspective)
```
Design implementation approach for: {TASK}

Based on exploration findings: {EXPLORER_SUMMARY}

Focus on:
- Script structure and organization
- Function decomposition
- Dependencies between components
- Reusable patterns from codebase

Output implementation design to ~/.claude/bash-workflow/plan-implementation.md
```

### Plan Agent (Testing Perspective)
```
Design test strategy for: {TASK}

Based on exploration findings: {EXPLORER_SUMMARY}

Focus on:
- Behaviors to verify with bats tests
- Edge cases: empty input, missing files, bad permissions, signals
- Error paths and failure modes
- Platform-specific test considerations

Output test strategy to ~/.claude/bash-workflow/plan-testing.md
```

### Plan Agent (Security Perspective)
```
Analyze security considerations for: {TASK}

Based on exploration findings: {EXPLORER_SUMMARY}

Focus on:
- Input validation requirements
- Permission handling (file modes, sudo usage)
- Secret handling (no hardcoded secrets, use env vars)
- Command injection risks
- Unsafe patterns to avoid

Output security analysis to ~/.claude/bash-workflow/plan-security.md
```

### Plan Agent (DevOps Perspective)
```
Analyze operational considerations for: {TASK}

Based on exploration findings: {EXPLORER_SUMMARY}

Focus on:
- macOS vs Linux compatibility (BSD vs GNU tools)
- CI/CD integration considerations
- Deployment and installation
- Dependency requirements (external commands needed)
- Idempotency for repeated runs

Output DevOps analysis to ~/.claude/bash-workflow/plan-devops.md
```

### bash-script-architect Agent (per-file)
```
Implement changes to: {FILE_PATH}

Task context:
{OVERALL_TASK_DESCRIPTION}

Unified plan summary:
{UNIFIED_PLAN_SUMMARY}

Scope for this file:
{FILE_SPECIFIC_CHANGES}

Context from exploration:
- Group: {GROUP_NUM} (files in this group execute in parallel)
- Related files in same group: {SIBLING_FILES}
- Interface contracts to implement: {INTERFACES}

Requirements:
- Follow bash-script-architect standards (set -euo pipefail, quoted variables)
- Include function documentation headers
- Handle errors with meaningful messages
- DO NOT write tests (bash-tdd-architect handles this)
- If defining interfaces other files depend on, make them clear

Output: Write implementation for {FILE_PATH} only
```

### bash-tdd-architect Agent (per-feature)
```
Write bats tests for feature: {FEATURE_NAME}

Behaviors to verify:
{BEHAVIOR_LIST}

Files involved: {FILE_LIST}

Security considerations from plan:
{SECURITY_NOTES}

Requirements:
- Design tests from SPEC, not by reading implementation
- Use bats-core with bats-assert and bats-support
- Cover happy paths, edge cases, and error conditions
- Test on both macOS and Linux if platform-specific code
- One test file per feature: test_{feature_slug}.bats

Test file location: test/ or match existing project convention

Output: Write comprehensive bats test file for this feature
```

### devops-infra-lead Agent (Reviewer)
```
Review implementation for: {TASK}

Files changed: {FILE_LIST}

Review using the Bash Code Review Checklist and Bats Test Review Checklist.

Check code for:
- set -euo pipefail usage
- Quoted variable expansions
- Error handling completeness
- Cross-platform compatibility
- Shellcheck compliance

Check tests for:
- Behavior-focused (not implementation-focused)
- Edge case coverage
- Test independence
- Meaningful assertions

Return verdict:
- [PASS]: Ready to commit
- [FAIL]: List test failures and critical issues
- [NEEDS_CHANGES]: List issues requiring fixes
```

### bash-test-runner Agent
```
Run shellcheck and bats tests for the project.

Return:
- [PASS] if all shellcheck clean and tests pass
- [FAIL]: [list of shellcheck errors and test failures]
- [ERROR]: [setup issues like missing bats]
```

### git-ops Agent
```
Commit and push: [summary of what was implemented]
```

## Verdict Processing

```python
if devops_lead.verdict == "[PASS]" and test_runner.result == "[PASS]":
    launch_git_ops()
elif devops_lead.verdict == "[FAIL]" or test_runner.result.startswith("[FAIL]"):
    # Synthesize feedback - don't copy-paste full output
    # Extract specific issues as bullet points
    retry_wave_2(changes=synthesized_feedback)
elif devops_lead.verdict == "[NEEDS_CHANGES]":
    # Minor issues - can often be fixed with direct edits
    fix_issues_or_retry_wave_2()
```

## Context Curation

You are the ORCHESTRATOR. Your job is to **distill and route information**—not to dump raw context.

| Agent | Needs | Does NOT need |
|-------|-------|---------------|
| Explore | Task description | Previous iteration history |
| Plan agents | Explorer findings, task context | Other plan perspectives (run in parallel) |
| bash-script-architect (per-file) | Task context, unified plan, file-specific scope | Other files' details, test info |
| bash-tdd-architect (per-feature) | Feature behaviors, security notes, file locations | Implementation details |
| devops-infra-lead | Files changed, summary of what was done | Full explorer output |
| bash-test-runner | Nothing beyond "run tests" | Any context |
| git-ops | Brief summary for commit message | Plan, feedback history |

**When passing iteration feedback:** Synthesize, don't copy-paste. Extract specific issues as bullet points.

## Todo Template

```
1. Create task breakdown for {FEATURE}
2. Wave 1a: Launch Explore agent (dependency graph + features)
3. Wave 1b: Launch 4 Plan agents in parallel
4. Synthesize unified plan from 4 perspectives
5. Wave 2 Group 1: Launch [N architects + M testers] for group 1 (TOGETHER)
6. Wave 2 Group 2+: Repeat for remaining groups (sequential)
7. Wave 3: Launch devops-infra-lead + bash-test-runner
8. Process verdict
9. Complete workflow / iterate if needed
10. Git operations (if approved)
```

## Output Directory

`~/.claude/bash-workflow/` for intermediate artifacts:
- `explorer-findings.md`
- `plan-implementation.md`
- `plan-testing.md`
- `plan-security.md`
- `plan-devops.md`
- `unified-plan.md`
- `review.md`

## Anti-Patterns to Avoid

1. **Don't launch script-architects and tdd-architects separately** - always in the same message
2. **Don't let tdd-architect see implementation first** - tests come from spec
3. **Don't skip the 4 Plan perspectives** - all 4 are required for Wave 1b
4. **Don't copy-paste full agent outputs** - synthesize to bullet points
5. **Don't skip devops-infra-lead review** even for "simple" changes
6. **Don't proceed to git-ops** without [PASS] from both reviewer AND test-runner
7. **Don't ignore test review** - reward hacking is a real failure mode
8. **Don't use Edit/Write for substantive changes** - delegate to bash-script-architect
9. **Don't bypass review with direct edits** - even "quick fixes" need quality gates
10. **Don't skip dependency analysis** - parallel writers can create conflicts
11. **Don't spawn tdd-architect per file** - tests are by feature, not file
12. **Don't proceed to group N+1** until group N completes

## Example: Multi-File Task

Task: "Add backup functionality with rotation to the sync scripts"

**Wave 1a - Explorer output:**
```
## Files
| Path | Action | Group | Notes |
|------|--------|-------|-------|
| scripts/lib/backup.sh | create | 1 | Backup functions (no deps) |
| scripts/sync.sh | modify | 2 | Add backup before sync |
| scripts/cleanup.sh | create | 2 | Rotation logic |

## Features
### Feature: Backup Creation
- Files: [scripts/lib/backup.sh, scripts/sync.sh]
- Behaviors:
  - Create timestamped backup before sync
  - Skip backup if no changes detected

### Feature: Backup Rotation
- Files: [scripts/cleanup.sh]
- Behaviors:
  - Keep last N backups (configurable)
  - Remove oldest when limit exceeded
```

**Wave 1b - 4 Plan agents (single message with 4 Task calls):**
```
Plan (Implementation): Focus on script structure
Plan (Testing): Edge cases for backup/rotation
Plan (Security): Permission handling for backup files
Plan (DevOps): Cross-platform tar/date commands
```

**Wave 2 execution:**
```
Group 1 (single message with 2 Task calls):
  - bash-script-architect: scripts/lib/backup.sh
  - bash-tdd-architect: "Backup Creation" feature
  [wait for completion]

Group 2 (single message with 4 Task calls):
  - bash-script-architect: scripts/sync.sh
  - bash-script-architect: scripts/cleanup.sh
  - bash-tdd-architect: "Backup Rotation" feature
  - bash-tdd-architect: "Sync Integration" feature (if identified)
  [wait for completion]
```

**Wave 3 (single message with 2 Task calls):**
```
  - bash-test-runner: Run shellcheck + bats
  - devops-infra-lead: Review all code and tests
  [wait for both, check verdicts]
```

## Success Criteria

The workflow is complete when ALL are true:
1. devops-infra-lead returned `[PASS]`
2. bash-test-runner returned `[PASS]` (shellcheck clean, bats pass)
3. git-ops confirmed commit successful

**There is NO shortcut.** Even if the change is trivial, the gate is: [PASS] + [PASS] = proceed.

## Output Format

Present final summary with:
- Implementation files (absolute paths)
- Test files (absolute paths)
- Review verdict
- Next steps if needed
