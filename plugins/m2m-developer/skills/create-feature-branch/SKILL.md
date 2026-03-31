---
name: create-feature-branch
description: Create a fresh feature or fix branch from the latest default branch. Use when user wants to start new work on a clean branch, e.g. "create a branch for X", "start a new feature branch", "new branch for fixing Y".
argument-hint: "[branch-name or description]"
allowed-tools: Bash(git *), AskUserQuestion
disable-model-invocation: true
---

# Create a Fresh Feature Branch

Create a new branch from an up-to-date default branch, ready for development.

## Step 1: Check for Uncommitted Changes

```bash
git status
```

If there are uncommitted changes (staged, unstaged, or untracked), warn the user and ask how to proceed. Suggest using the `move-changes-to-branch` skill if they want to bring changes along.

## Step 2: Determine Branch Name

If `$ARGUMENTS` is provided, use it as or derive the branch name from it. Otherwise, ask the user what the branch is for.

Use conventional prefixes: `feat/`, `fix/`, `chore/`, `refactor/`, `docs/`.

## Step 3: Detect the Default Branch

```bash
DEFAULT_BRANCH=$(git remote show origin | grep 'HEAD branch' | awk '{print $NF}')
```

If `git remote show origin` fails (no remote configured), ask the user which branch to base off of.

## Step 4: Sync the Default Branch

```bash
git checkout $DEFAULT_BRANCH
git pull origin $DEFAULT_BRANCH
```

## Step 5: Create and Switch to the New Branch

```bash
git checkout -b <new-branch-name>
```

## Step 6: Report

Show:
- New branch name
- Base branch and commit it was created from (`git log --oneline -1`)
