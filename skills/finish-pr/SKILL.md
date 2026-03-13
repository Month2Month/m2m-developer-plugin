---
name: finish-pr
description: Finalize a pull request by fixing test/lint/build failures, running CI checks, updating PR description, and marking ready for review. Auto-detects project type (Rails, Node, Python, Go, Rust, etc.). Handles the feedback loop for PRs where code is written but CI is failing or docs are incomplete. Use when user asks to "finish this PR", "complete the PR", "fix PR failures", or "update PR description".
argument-hint: [pr-number-or-branch]
allowed-tools: Bash(gh:*), Bash(git:*), Bash(bundle:*), Bash(rails:*), Bash(rake:*), Bash(npm:*), Bash(npx:*), Bash(yarn:*), Bash(pnpm:*), Bash(python:*), Bash(pytest:*), Bash(pip:*), Bash(cargo:*), Bash(go:*), Bash(make:*), Bash(curl:*), Read, Grep, Write, Edit, Glob, Task
disable-model-invocation: false
---

# Finish Pull Request Workflow

Complete a pull request by fixing CI failures, running checks, and updating documentation. Auto-detects project type.

## When to Use This Skill

Use this skill when:
- PR exists but has failing CI checks (tests, linting, types, build)
- PR description needs to be updated with comprehensive documentation
- Draft PR needs to be finalized and marked ready for review
- User asks to "finish", "complete", or "fix" a PR

## Step 1: Fetch and Understand the PR

Fetch the PR details to understand what was implemented:

```bash
# If branch name provided, find the PR
gh pr list --head $ARGUMENTS --json number,title,body,isDraft,headRefName

# Or if PR number provided
gh pr view $ARGUMENTS --json number,title,body,isDraft,headRefName,url,commits
```

Extract and summarize:
- What issue(s) this PR addresses (look for "Fixes #X" or "Closes #X")
- What changes were made (review commits)
- Current PR description completeness
- Whether it's a draft PR

If needed, read the linked issue for full context:
```bash
gh issue view <issue-number> --json title,body,labels,comments
```

## Step 2: Detect Project Type and Review Changes

Detect the project stack by checking for key files:

```
Gemfile          → Ruby/Rails     → bundle exec rspec, rubocop
package.json     → Node/JS/TS     → npm test, eslint, tsc
pyproject.toml   → Python         → pytest, ruff/flake8
Cargo.toml       → Rust           → cargo test, cargo clippy
go.mod           → Go             → go test, golangci-lint
Makefile         → Check targets  → make test, make lint
```

Also check:
- `CLAUDE.md` or `README.md` for project conventions
- `.github/workflows/` for CI configuration
- Linter configs for what rules are enforced

Then review the PR's changes:

```bash
# Fetch latest and checkout the PR branch
git fetch origin
git checkout <branch-name>
git pull origin <branch-name>

# Detect default branch
DEFAULT_BRANCH=$(git remote show origin | grep 'HEAD branch' | awk '{print $NF}')

# Review what changed
git diff $DEFAULT_BRANCH...HEAD --stat
git log $DEFAULT_BRANCH..HEAD --oneline
```

Read the changed files to understand the implementation.

## Step 3: Check Most Recent CI Results First

**Before running any tests locally**, pull the latest CI results from GitHub to understand what's already failing:

```bash
# Get the most recent CI check runs for this PR
gh pr checks $ARGUMENTS

# Get detailed workflow run logs if checks are failing
gh run list --branch <branch-name> --limit 3
```

If there are failing checks:

```bash
# Get the run ID of the most recent failed run
RUN_ID=$(gh run list --branch <branch-name> --limit 1 --json databaseId,status,conclusion -q '.[0].databaseId')

# Download and inspect the failure logs
gh run view $RUN_ID --log-failed
```

**Analyze CI failures before running anything locally:**
- Read the failed test names and error messages from the CI logs
- Identify which files failed and on which lines
- Note any environment-specific failures (e.g., missing services, timeouts)
- Determine if failures are related to the PR changes or pre-existing flaky tests

**Based on CI results:**
- All CI checks pass → Skip to Step 5 (just update PR description)
- CI checks failing → Use the failure output to guide your fix in Step 4 (skip running tests locally until you've made a fix attempt)
- No CI runs yet or CI still running → Proceed to Step 3b to run tests locally

## Step 3b: Run CI Checks Locally (only if no CI results available)

Only run checks locally if CI hasn't run yet or you need to verify a fix. Run the project's CI checks based on the detected stack. Prefer `Makefile` targets if available.

**Linting:**
```bash
# Ruby: bundle exec rubocop
# JS/TS: npm run lint
# Python: ruff check . / flake8
# Go: golangci-lint run
# Rust: cargo clippy
# Or: make lint
```

**Tests:**
```bash
# Ruby: bundle exec rspec (relevant specs)
# JS/TS: npm test
# Python: pytest
# Go: go test ./...
# Rust: cargo test
# Or: make test
```

**Type checking / Build (if applicable):**
```bash
# TypeScript: npx tsc --noEmit / npm run build
# Python: mypy / pyright
# Rust: cargo build
# Go: go build ./...
```

Focus on tests relevant to the PR's changes for large test suites.

**Track the results:**
- All checks pass → Skip to Step 5
- Any checks fail → Proceed to Step 4

## Step 4: Fix Failures with Retry Logic

**IF ANY CHECKS FAIL:**

### Retry Loop (Maximum 5 Attempts)

For each failed attempt (up to 5 total):

1. **Analyze the Failure**
   - Read the error output carefully
   - Categorize: lint error, type error, test failure, build error, runtime error
   - Identify root cause and which files need changes

2. **Create a Fix Plan**
   - Outline specific changes needed
   - Prioritize: lint/format → type errors → test setup → logic errors → edge cases
   - Consider if the error indicates a flaw in the original approach

3. **Implement the Fix**
   - Make targeted changes to address specific errors
   - Don't rewrite working code - only fix what's broken

4. **Re-run Checks**
   - Run the same checks that failed
   - Track attempt number: "Attempt X/5"
   - Document what was fixed

5. **Evaluate Results**
   - All checks pass → Proceed to Step 5
   - Still failing AND attempt < 5 → Continue to next retry
   - Still failing AND attempt = 5 → Escalate (see below)

### After 5 Failed Attempts - Escalate

If checks still fail after 5 attempts:

1. **Commit current state with detailed context**:
   ```bash
   git add -A
   git commit -m "$(cat <<'EOF'
   WIP: Additional fix attempts for PR

   Attempted to fix CI failures with 5 retry attempts.
   Checks still failing - requires human review.

   Attempts made:
   1. [What was tried in attempt 1]
   2. [What was tried in attempt 2]
   3. [What was tried in attempt 3]
   4. [What was tried in attempt 4]
   5. [What was tried in attempt 5]

   Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
   EOF
   )"

   git push
   ```

2. **Update PR with escalation comment**:
   ```bash
   gh pr comment $ARGUMENTS --body "$(cat <<'EOF'
   ⚠️ **Automated fix attempted but CI checks still failing after 5 attempts**

   ## Retry Attempts
   1. **Attempt 1**: [What was tried and what failed]
   2. **Attempt 2**: [What was tried and what failed]
   3. **Attempt 3**: [What was tried and what failed]
   4. **Attempt 4**: [What was tried and what failed]
   5. **Attempt 5**: [What was tried and what failed]

   ## Current Failures
   ```
   [Most recent error output]
   ```

   ## Analysis
   [Analysis of why fixes didn't work and what might be needed]
   EOF
   )"
   ```

3. **Inform the user** and **stop the workflow**.

**IF ALL CHECKS PASS:** Proceed to Step 5.

## Step 5: Commit and Push Fixes

Only if all checks passed:

```bash
git add -A
git commit -m "$(cat <<'EOF'
Fix CI failures and finalize implementation

[Describe what was fixed - be specific]
- Fixed [specific issue 1]
- Updated [specific issue 2]

All checks now passing.

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"

git push
```

## Step 6: Update PR Description

Create a comprehensive PR description based on what was changed:

```bash
gh pr edit $ARGUMENTS --body "$(cat <<'EOF'
## Issue
Fixes #X

## Summary
[High-level overview of what was implemented]

## Changes
- [Change 1 with file reference]
- [Change 2 with file reference]

## Testing
- [x] All relevant tests passing
- [x] Linting/formatting checks passing
- [x] Type checking passing (if applicable)
- [x] No regressions

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Refactoring
- [ ] Documentation update

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

Adapt the description to include project-specific sections as appropriate:
- **API changes**: Document new/modified endpoints with request/response examples
- **Database changes**: List migrations, new tables/columns
- **UI changes**: Describe visual changes, include screenshots if relevant
- **Config changes**: Note any environment variable or config file changes

## Step 7: Update PR Title

If the PR title has "WIP:" prefix and all checks pass, remove it:

```bash
CURRENT_TITLE=$(gh pr view $ARGUMENTS --json title -q .title)
NEW_TITLE=$(echo "$CURRENT_TITLE" | sed 's/^WIP: //')
if [ "$NEW_TITLE" != "$CURRENT_TITLE" ]; then
  gh pr edit $ARGUMENTS --title "$NEW_TITLE"
fi
```

## Step 8: Mark PR Ready for Review

If the PR is a draft and all checks pass:

```bash
gh pr ready $ARGUMENTS
```

## Step 9: Provide Summary

Report to the user:

```
✅ PR successfully completed!

**Changes:**
- Fixed [X test failures / lint issues / type errors / etc.]
- Updated PR description with documentation
- All CI checks passing
- PR marked ready for review

**PR URL:** [link]

**Next Steps:**
- PR is ready for code review
- All checks passing
```

## Important Notes

- **Auto-detect project type** - don't assume the stack, detect it from project files
- **Respect project conventions** - check CLAUDE.md, CONTRIBUTING.md, existing patterns
- **Check CI config** - `.github/workflows/` tells you exactly what CI runs
- **Use the right tools** - match the project's package manager, test runner, linter
- **Run relevant tests** - don't run the entire suite for focused changes in large projects
- **Retry up to 5 times** with targeted fixes before escalating
- **Keep fix commits focused** - don't refactor unrelated code
- **Match commit style** - check `git log` for the project's conventions
