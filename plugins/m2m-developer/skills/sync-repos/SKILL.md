---
name: sync-repos
description: Check out default branches and pull latest for all repos in the current workspace
allowed-tools: Bash(git *), Bash(ls */.git), Bash(test -d .git *)
---

Sync git repos to their default branches.

## Steps

1. Check if the current working directory is itself a git repo (`test -d .git`)
   - **If yes**: sync this single repo and stop
   - **If no**: look one level down for git repos (`ls -d */.git` to find `<dir>/.git` entries)
2. For each repo, determine the default branch: `git remote show origin | grep 'HEAD branch'`
3. For each repo:
   - `git checkout <default-branch>`
   - `git pull origin <default-branch>`
4. If there are multiple repos, run them in parallel since they are independent

After completing, show a summary table with columns: Repo, Branch, Status (e.g. "up to date", "X files changed", or error message).
