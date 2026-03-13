---
name: worktree
description: This skill should be used when the user asks to "create a worktree", "set up a worktree", "work in a new branch", "start a new task branch", "worktree for issue", or wants to work on a separate branch without disrupting their current working directory. Manages git worktrees for parallel development.
disable-model-invocation: true
argument-hint: "[branch-name or task-description]"
---

# Git Worktree Setup

You are setting up a git worktree so the user can work on a task in an isolated branch without disrupting their current working directory.

## Step 1: Validate Git Repository

Confirm the current directory is inside a git repository. If not, stop and inform the user.

## Step 2: Determine the Branch

Based on the user's input (`$ARGUMENTS`), decide whether to use an existing branch or create a new one:

### Option A: Find an Existing Branch

Search local and remote branches for a match:

```
git branch -a
```

- If the user provided a branch name and it exists (locally or on a remote), use that branch.
- If it exists only on a remote, create a local tracking branch from it.

### Option B: Create a New Branch

If no matching branch exists, or the user described a task rather than a branch name:

- Derive a short, descriptive branch name from the task description (e.g., `fix/login-redirect`, `feat/user-avatar-upload`).
- Use conventional prefixes: `feat/`, `fix/`, `chore/`, `refactor/`, `docs/` as appropriate.
- Confirm the branch name with the user before creating it.
- Create the branch from the current default branch (main/master), making sure it's up to date:

```
git fetch origin
git branch <new-branch> origin/<default-branch>
```

## Step 3: Create the Worktree

All worktrees go in `~/dev/.worktrees/`. Create the directory if it doesn't exist.

Use the repo name and branch to build the worktree path:

```
~/dev/.worktrees/<repo-name>/<branch-name>
```

For example, if the repo is `my-app` and the branch is `feat/new-login`, the worktree goes at `~/dev/.worktrees/my-app/feat/new-login`.

```
mkdir -p ~/dev/.worktrees/<repo-name>
git worktree add ~/dev/.worktrees/<repo-name>/<branch-name> <branch>
```

If there's a naming conflict, inform the user and ask how to proceed.

## Step 4: Change Working Directory

**This is critical.** Change the working directory to the new worktree:

```
cd ~/dev/.worktrees/<repo-name>/<branch-name>
```

Confirm you are now in the worktree directory before proceeding.

## Step 5: Environment Setup

Check if the project needs environment setup by looking for common indicators:

1. **Node.js** (`package.json`): Run `npm install` or `yarn install` or `pnpm install` (match the lockfile).
2. **Python** (`requirements.txt`, `pyproject.toml`, `Pipfile`): Set up venv and install dependencies.
3. **Ruby** (`Gemfile`): Run `bundle install`.
4. **Rust** (`Cargo.toml`): Run `cargo build`.
5. **Go** (`go.mod`): Run `go mod download`.
6. **Environment files** (`.env.example`): Copy to `.env` if `.env` doesn't exist, and alert the user they may need to fill in secrets.

Ask the user before running any install commands. If the project type is unclear, ask what setup is needed.

## Step 6: Confirm and Start Working

Summarize what was done:

- Branch name and whether it was newly created or existing
- Worktree location
- Any environment setup performed

Then ask the user what they'd like to work on in this worktree, or begin working on the task they originally described.

## Important Notes

- Never run `git worktree add` with `--force` unless the user explicitly requests it.
- If the worktree path already exists, inform the user and ask how to proceed rather than overwriting.
- If the user wants to clean up later, they can run `git worktree remove <path>` from the main repo.
