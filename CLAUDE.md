# m2m-developer-plugin

Claude Code plugin shipping shared developer skills for the M2M team.

## Releasing a new version

When adding/editing a skill or otherwise changing plugin contents, bump the version in **both** files in the same commit:

- `.claude-plugin/marketplace.json` → `plugins[0].version`
- `plugins/m2m-developer/plugin.json` → `version`

`/plugin` reads `marketplace.json` to decide whether an update is available; the install cache directory is keyed on `plugin.json`'s version. Bumping only one of them silently fails:

- Only `plugin.json` bumped → `/plugin` reports "already at the latest version" and never updates the cache.
- Only `marketplace.json` bumped → install proceeds but cache directory name doesn't roll, so stale skill files can linger.

After pushing to `main`, clients run `/plugin` → update m2m-developer, then `/reload-plugins`.

## Skill structure

Each skill lives under `plugins/m2m-developer/skills/<skill-name>/SKILL.md` with YAML frontmatter (`name`, `description`) followed by instructions. Test locally with `claude --plugin-dir .` before opening a PR.
