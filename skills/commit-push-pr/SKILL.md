---
name: commit-push-pr
description: Sync default branch, create feature branch, commit staged changes, push, and open a pull request. Use when user asks to "commit and PR", "push and create PR", or "send this up for review".
argument-hint: "[branch-name or description]"
allowed-tools: Bash(git *), Bash(gh *)
disable-model-invocation: true
---

# Commit, Push, and Create PR

Automate the full flow from local changes to an open pull request.

## Step 1: Assess Current State

```bash
git status
git diff --stat
git log --oneline -5
```

Understand what's changed and what needs to be committed.

## Step 2: Determine the Default Branch

```bash
DEFAULT_BRANCH=$(git remote show origin | grep 'HEAD branch' | awk '{print $NF}')
```

## Step 3: Create Feature Branch (if needed)

If the user is on the default branch, create a new feature branch:

- Use the `$ARGUMENTS` as the branch name or derive one from the changes
- Use conventional prefixes: `feat/`, `fix/`, `chore/`, `refactor/`, `docs/`
- Branch from the latest default branch:

```bash
git fetch origin
git checkout -b <branch-name>
```

If already on a feature branch, stay on it.

## Step 4: Stage and Commit

Review the changes and create a meaningful commit:

```bash
# Stage relevant files (prefer specific files over git add -A)
git add <files>

# Commit with a descriptive message
git commit -m "$(cat <<'EOF'
[type]: [concise description]

[Optional body explaining why, not what]

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

- Match the repo's existing commit message style (check `git log`)
- Use conventional commits if the project follows that convention

## Step 5: Push

```bash
git push -u origin HEAD
```

## Step 6: Create Pull Request

```bash
gh pr create --title "[concise title]" --body "$(cat <<'EOF'
## Summary
- [Key change 1]
- [Key change 2]

## Test plan
- [ ] [How to verify this works]

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

- Link related issues with "Fixes #X" or "Relates to #X"
- Keep the title under 70 characters
- Include a test plan

## Step 7: Report

Provide the PR URL and a brief summary of what was pushed.
