# M2M Developer Plugin

Claude Code plugin with shared developer skills for the M2M team.

## Skills

| Skill | Command | Description |
|-------|---------|-------------|
| **fix-issue** | `/m2m-developer:fix-issue 42` | Fix a GitHub issue end-to-end: fetch, branch, implement, test, PR |
| **finish-pr** | `/m2m-developer:finish-pr 123` | Fix failing CI, update description, mark PR ready for review |
| **commit-push-pr** | `/m2m-developer:commit-push-pr` | Stage, commit, push, and open a PR in one shot |
| **sync-repos** | `/m2m-developer:sync-repos` | Pull latest on default branches for all repos in workspace |
| **worktree** | `/m2m-developer:worktree feat/new-thing` | Create an isolated git worktree for parallel development |

## Installation

### Option A: Install from GitHub (recommended for team)

1. Add the marketplace (one-time setup):
   ```
   /plugin marketplace add m2m-team/m2m-developer-plugin
   ```

2. Install the plugin:
   ```
   /plugin install m2m-developer@m2m-team
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
