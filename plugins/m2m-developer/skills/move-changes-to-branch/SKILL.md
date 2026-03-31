---
name: move-changes-to-branch
description: Move uncommitted work (staged, unstaged, and untracked files) from the current branch onto a new feature branch created from the latest default branch. Use when user has changes on the wrong branch or wants to start a clean feature branch with current work.
argument-hint: "[new-branch-name]"
allowed-tools: Bash(git *), AskUserQuestion
disable-model-invocation: true
---

# Move Changes to a New Feature Branch

Move uncommitted work (staged, unstaged, and untracked files) from the current branch onto a new feature branch created from the latest default branch.

## Step 1: Verify Changes Exist

```bash
git status
```

If the working tree is clean with no untracked files, stop and tell the user there's nothing to move.

## Step 2: Show What Will Be Moved

Summarize staged changes, unstaged changes, and untracked files.

If `$ARGUMENTS` is empty, ask the user for the new branch name. Use conventional prefixes: `feat/`, `fix/`, `chore/`, `refactor/`, `docs/`.

## Step 3: Detect the Default Branch

```bash
DEFAULT_BRANCH=$(git remote show origin | grep 'HEAD branch' | awk '{print $NF}')
```

If `git remote show origin` fails (no remote configured), ask the user which branch to base off of.

## Step 4: Stash Everything

```bash
git stash push --include-untracked -m "move-changes-to-branch: temp stash"
```

## Step 5: Sync the Default Branch

```bash
git checkout $DEFAULT_BRANCH
git pull origin $DEFAULT_BRANCH
```

## Step 6: Create and Switch to the New Branch

```bash
git checkout -b <new-branch-name>
```

## Step 7: Restore Stashed Changes

```bash
git stash pop
```

**If merge conflicts occur:** Do NOT auto-resolve. List the conflicted files and ask the user how to proceed.

## Step 8: Stage and Commit

```bash
# Stage all changes including untracked files
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

## Step 9: Report

Show:
- Previous branch
- New branch name
- Commit hash
- Files changed
