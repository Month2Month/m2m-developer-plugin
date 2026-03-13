# M2M Developer Plugin

Claude Code plugin with shared developer skills for the M2M team.

## Skills

| Skill | Command | Description |
|-------|---------|-------------|
| **finish-pr** | `/m2m-developer:finish-pr 123` | Fix failing CI, update description, mark PR ready for review |
| **fix-issue** | `/m2m-developer:fix-issue 42` | Fix a GitHub issue end-to-end: fetch, branch, implement, test, PR |
| **commit-push-pr** | `/m2m-developer:commit-push-pr` | Stage, commit, push, and open a PR in one shot |
| **sync-repos** | `/m2m-developer:sync-repos` | Pull latest on default branches for all repos in workspace |
| **worktree** | `/m2m-developer:worktree feat/new-thing` | Create an isolated git worktree for parallel development |

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

### worktree

You want to tackle a feature or issue without disrupting your current working directory or other Claude sessions:

```
/m2m-developer:worktree fix/login-bug
```

Creates an isolated git worktree branched off the default branch. Useful when you're juggling multiple things and don't want context bleed between tasks.

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
