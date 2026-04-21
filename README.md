# M2M Developer Plugin

Claude Code plugin with shared developer skills for the M2M team.

## Skills

| Skill | Command | When to use | Description |
|-------|---------|-------------|-------------|
| **finish-pr** | `/m2m-developer:finish-pr 123` | PR pipeline is failing, fix it locally instead of @claude on GH | Fix failing CI, update description, mark PR ready for review |
| **fix-issue** | `/m2m-developer:fix-issue 42` | Well-specced issue, go from zero to PR | Fix a GitHub issue end-to-end: fetch, branch, implement, test, PR |
| **commit-push-pr** | `/m2m-developer:commit-push-pr` | Local changes ready to go, need branch + commit + PR in one shot | Sync default branch, create feature branch, commit, push, open PR |
| **sync-repos** | `/m2m-developer:sync-repos` | Parent dir with many repos that need to be up to date | Pull latest on default branches for all repos in workspace |
| **move-changes-to-branch** | `/m2m-developer:move-changes-to-branch feat/thing` | Uncommitted changes on wrong branch, need to move them | Stash work, sync default branch, create feature branch, restore and commit |
| **create-feature-branch** | `/m2m-developer:create-feature-branch feat/thing` | Starting new work, need a clean branch | Create a fresh feature branch from latest default branch |
| **worktree** | `/m2m-developer:worktree feat/new-thing` | Tackle a feature/issue without disrupting other sessions | Create an isolated git worktree branched off default |
| **feature-release-notes** | `/m2m-developer:feature-release-notes` | Announce a feature (or weekly roundup) to the company | Find the merged PRs across the M2M org, summarize in the user's voice, save to /tmp |

## When to Use Each Skill

### finish-pr

You see a PR — yours or one Claude made — and the pipeline is failing. Instead of leaving a `@claude` comment on GitHub and waiting, just clone/cd into the repo locally and run:

```
/m2m-developer:finish-pr 123
```

Claude runs locally, so it can actually execute the tests, see what's broken, fix it, and push — all without the back-and-forth of GitHub comment-driven fixes.

### fix-issue

You have a well-specced GitHub issue and want to go from zero to PR. Just point Claude at the issue number:

```
/m2m-developer:fix-issue 42
```

It fetches the issue, creates a branch, implements the fix, writes tests, runs CI checks, and opens a PR. Best used when the issue has enough detail for Claude to work autonomously.

### commit-push-pr

You've been working locally — changes are sitting uncommitted, maybe you're on `main`, maybe on a random branch. Doesn't matter. This skill handles the full flow:

```
/m2m-developer:commit-push-pr
```

It syncs the default branch, creates an appropriately named feature branch, commits your changes, pushes, and opens a PR. Works no matter where you are in the git tree.

### move-changes-to-branch

You've been hacking on `main` or a stale branch and realize your changes belong on a proper feature branch:

```
/m2m-developer:move-changes-to-branch feat/new-search
```

It stashes everything (including untracked files), syncs the default branch, creates a new branch, restores your work, and commits it. If you don't provide a branch name, it'll ask.

### create-feature-branch

You're about to start new work and want a clean branch off the latest default:

```
/m2m-developer:create-feature-branch fix/login-redirect
```

Syncs the default branch and creates a new branch from it. If you have uncommitted changes, it warns you and suggests `move-changes-to-branch` instead.

### worktree

You want to tackle a feature or issue without disrupting your current working directory or other Claude sessions:

```
/m2m-developer:worktree fix/login-bug
```

Creates an isolated git worktree branched off the default branch. Useful when you're juggling multiple things and don't want context bleed between tasks.

### feature-release-notes

You shipped a feature (or a batch of features this week) and want to write it up for the company — not just engineering. Describe the feature in plain language:

```
/m2m-developer:feature-release-notes
```

Finds the merged PRs across the Month2Month org that implemented the feature, filters out the tangential ones, and drafts a Slack/email-ready announcement in the user's voice — `Hi team, I'm excited to announce…`. Supports single-feature announcements and multi-feature weekly roundups (emoji-headed product-area sections). PR references are kept in a delete-before-sending footer. Saves to `/tmp/release-notes-<slug>-<date>.md`.

### sync-repos

Pull latest on all repos in your workspace at once:

```
/m2m-developer:sync-repos
```

Most useful when you're working from a parent directory with many child repos that all need to be synced before you start a new prompt. Also handy when you just need fresh context and up-to-date repos without running the git commands yourself.

## Installation

### Option A: Install from GitHub (recommended for team)

1. Add the marketplace (one-time setup):
   ```
   /plugin marketplace add Month2Month/m2m-developer-plugin
   ```

2. Install the plugin:
   ```
   /plugin install m2m-developer@Month2Month
   ```

### Option B: Install locally for testing

```bash
claude --plugin-dir /path/to/m2m-developer-plugin
```

## Adding New Skills

1. Create a new directory under `skills/` with the skill name
2. Add a `SKILL.md` file with YAML frontmatter and instructions
3. Test locally with `claude --plugin-dir .`
4. Open a PR for team review
