---
name: fix-issue
description: Analyze and fix GitHub issues end-to-end. Auto-detects project type (Rails, Node, Python, Go, Rust, etc.), fetches issue details, implements fixes, writes tests, runs CI checks, and creates PRs. If CI checks fail, retries up to 5 times. After 5 failed attempts, creates a draft PR and escalates. Use when user asks to fix or work on a GitHub issue.
argument-hint: [issue-number]
allowed-tools: Bash(gh:*), Bash(git:*), Bash(bundle:*), Bash(rails:*), Bash(rake:*), Bash(npm:*), Bash(npx:*), Bash(yarn:*), Bash(pnpm:*), Bash(python:*), Bash(pytest:*), Bash(pip:*), Bash(cargo:*), Bash(go:*), Bash(make:*), Bash(curl:*), Read, Grep, Write, Edit, Glob, Task
disable-model-invocation: false
---

# Fix GitHub Issue Workflow

Please analyze and fix GitHub issue #$ARGUMENTS following this complete workflow:

## Step 1: Fetch and Understand the Issue

Use `gh issue view $ARGUMENTS --json title,body,labels,comments` to get full issue details.

Extract and summarize:
- The problem description
- Acceptance criteria
- Expected vs actual behavior
- Any relevant labels (bug, feature, enhancement, etc.)
- Comments that provide additional context

## Step 2: Detect Project Type

Before proceeding, detect the project stack by checking for key files:

```
Gemfile          → Ruby/Rails     → use bundle exec rspec, rubocop
package.json     → Node/JS/TS     → use npm/yarn/pnpm test, eslint
pyproject.toml   → Python         → use pytest, ruff/flake8
requirements.txt → Python         → use pytest, ruff/flake8
Cargo.toml       → Rust           → use cargo test, cargo clippy
go.mod           → Go             → use go test, golangci-lint
Makefile         → Check targets  → use make test, make lint if available
```

Also check for:
- `CLAUDE.md` or `README.md` for project-specific conventions
- `.github/workflows/` for CI configuration (tells you exactly what CI runs)
- Existing test directories to understand test structure
- Linter configs (`.rubocop.yml`, `.eslintrc`, `ruff.toml`, `.golangci.yml`, etc.)

Store the detected stack info - you'll use it throughout the workflow.

## Step 3: Determine Default Branch and Create Feature Branch

```bash
# Detect the default branch
DEFAULT_BRANCH=$(git remote show origin | grep 'HEAD branch' | awk '{print $NF}')

# Update and branch from it
git checkout $DEFAULT_BRANCH && git pull origin $DEFAULT_BRANCH
git checkout -b fix/$ARGUMENTS-brief-description
```

Use `fix/` prefix for bugs and `feature/` prefix for new features or enhancements. Follow any branch naming conventions found in CLAUDE.md or contributing docs.

## Step 4: Investigate the Codebase

- Use Grep/Glob to find files mentioned in the issue
- Read the current implementation
- Check `CLAUDE.md` or `CONTRIBUTING.md` for project conventions
- Review existing tests for similar functionality
- Understand the architecture before making changes

## Step 5: Implement the Fix

- Make necessary code changes following the project's existing conventions
- Keep changes focused and minimal
- Match the code style of surrounding code
- Add comments only for non-obvious logic

## Step 6: Write/Update Tests

Write tests that:
- Verify the fix resolves the issue
- Cover edge cases mentioned in the issue
- Don't break existing functionality

Run only the relevant tests first to iterate quickly:

**Ruby/Rails:**
```bash
bundle exec rspec spec/path/to/relevant_spec.rb
```

**Node/JS/TS:**
```bash
npm test -- --testPathPattern=path/to/test
# or: npx jest path/to/test
# or: npx vitest run path/to/test
```

**Python:**
```bash
pytest tests/path/to/test_file.py -v
```

**Go:**
```bash
go test ./path/to/package/... -v -run TestName
```

**Rust:**
```bash
cargo test test_name -- --nocapture
```

## Step 7: Check CI Results and Run Checks

**First, check if there are existing CI results** (e.g., if you've already pushed):

```bash
# Check for any CI runs on the current branch
gh run list --branch $(git branch --show-current) --limit 3
```

If CI has run and there are failures:

```bash
# Get the most recent failed run ID
RUN_ID=$(gh run list --branch $(git branch --show-current) --limit 1 --json databaseId,status,conclusion -q '.[0].databaseId')

# Download and inspect the failure logs
gh run view $RUN_ID --log-failed
```

**Analyze CI failures before running checks locally** - use the CI output to understand what's failing and why.

**If no CI results are available**, run the project's CI checks locally. Prefer `Makefile` targets if available, otherwise use the detected stack commands:

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

**Type checking (if applicable):**
```bash
# TypeScript: npx tsc --noEmit
# Python: mypy / pyright
```

**Note**: For large test suites, focus on specs relevant to your changes rather than the entire suite unless changes are wide-reaching.

## Step 8: Handle Failures with Retry Logic

**IF ANY CHECKS FAIL:**

### Retry Loop (Maximum 5 Attempts)

For each failed attempt (up to 5 total):

1. **Analyze the Failure**
   - Read the error output carefully
   - Identify root cause (syntax error, logic error, missing dependency, type error, etc.)
   - Determine which files need modification

2. **Create a Fix Plan**
   - Outline specific changes needed
   - Prioritize: syntax/lint errors → type errors → test setup → logic errors → edge cases
   - Consider if the error indicates a flaw in the original approach

3. **Implement the Fix**
   - Make targeted changes to address the specific errors
   - Don't rewrite working code - only fix what's broken

4. **Re-run Checks**
   - Run the same checks that failed
   - Track attempt number (Attempt 1/5, 2/5, etc.)
   - Document what was fixed

5. **Evaluate Results**
   - If ALL checks pass → Proceed to Step 9
   - If checks still fail AND attempt < 5 → Continue to next retry
   - If checks still fail AND attempt = 5 → Escalate (see below)

### After 5 Failed Attempts - Escalate

If checks still fail after 5 attempts:

1. **Create a draft PR with detailed context**:
   ```bash
   git add -A
   git commit -m "$(cat <<'EOF'
   WIP: Fix #$ARGUMENTS - [brief description]

   Attempted fix with 5 retry attempts.
   CI checks still failing - requires human review.

   Attempts made:
   1. [What was tried in attempt 1]
   2. [What was tried in attempt 2]
   3. [What was tried in attempt 3]
   4. [What was tried in attempt 4]
   5. [What was tried in attempt 5]

   Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
   EOF
   )"

   git push -u origin HEAD
   ```

   Then create the draft PR:
   ```bash
   gh pr create --draft \
     --title "WIP: Fix #$ARGUMENTS - [brief description]" \
     --body "$(cat <<'EOF'
   ## Issue
   Fixes #$ARGUMENTS

   ## Changes
   [Describe what was changed]

   ## Status
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
   [Analysis of why the fixes didn't work and what might be needed]

   🤖 Generated with [Claude Code](https://claude.com/claude-code)
   EOF
   )"
   ```

2. **Inform the user** and **stop the workflow**.

**IF ALL CHECKS PASS:** Proceed to Step 9.

## Step 9: Create Production-Ready Commit

Only if all checks passed:

```bash
git add -A
git commit -m "$(cat <<'EOF'
Fix #$ARGUMENTS: [clear, descriptive summary]

[Detailed explanation of the fix]

Closes #$ARGUMENTS

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

## Step 10: Push and Create PR

```bash
git push -u origin HEAD

gh pr create \
  --title "Fix #$ARGUMENTS: [brief description]" \
  --body "$(cat <<'EOF'
## Issue
Fixes #$ARGUMENTS

## Changes
[Describe the changes made with file references]

## Testing
- [x] Relevant tests passing
- [x] No regressions in related tests
- [x] Linting/type checks passing
- [x] Edge cases covered

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

Provide the PR URL to the user.

## Important Notes

- **Auto-detect everything** - don't assume the stack, detect it from project files
- **Respect project conventions** - check CLAUDE.md, CONTRIBUTING.md, and existing code patterns
- **Use the right tools** - match the project's package manager, test runner, linter
- **Check CI config** - `.github/workflows/` tells you exactly what CI runs
- **Run relevant tests** - don't run the entire suite for focused changes
- **Retry up to 5 times** before escalating with a draft PR
- **Keep changes focused** - don't refactor unrelated code
- **Match commit style** - check `git log` for the project's commit message conventions
