---
name: feature-release-notes
description: Draft an internal cross-department announcement for a specific M2M feature, or a weekly/multi-feature roundup, by finding the relevant merged PRs across the Month2Month GitHub org and summarizing them in the user's voice (warm, direct, non-technical). Use when the user asks to "write release notes", "draft a feature announcement", "summarize what shipped for [feature]", "write up what we launched", "write a weekly roundup", "what shipped this week", or wants to communicate feature launches to the broader company. Outputs a Slack/email-ready markdown block and saves to /tmp.
argument-hint: [feature description]
allowed-tools: Bash(gh:*), Bash(date:*), Bash(mkdir:*), Read, Write, Grep, Glob
---

# Feature Release Notes

Find the merged PRs across the Month2Month GitHub org that implemented a specific feature, and turn them into an internal announcement that people outside engineering can read.

## When to use

The user has shipped (or is about to announce) a feature and wants a write-up for the rest of the company — customer success, operations, sales, finance, partners — not just engineers. They'll describe the feature in plain language; you go find the merged PRs that relate to it and synthesize a clear, non-technical summary.

Do **not** use this skill for:
- Per-repo `CHANGELOG.md` maintenance (wrong audience, too granular).
- Generic "what shipped this week" digests with no feature focus — this skill needs a specific feature to filter against.

## Operating model

- **Source of truth is GitHub, not the local filesystem.** Everything about repos and PRs comes from `gh` queries against the `Month2Month` org. Do not scan local directories or assume any repo is cloned.
- **Repo knowledge comes from `references/m2m-repos.md`** (bundled with this skill). Read that file first — it documents what each known repo is responsible for and has a domain-to-repos lookup table. For any repo you encounter that isn't in the reference file, fall back to `gh repo view Month2Month/<repo> --json description`.
- **The documented set is not exhaustive.** Features can ship in repos you've never heard of. Always list org repos via `gh` rather than assuming you know the full set.

## Required inputs

Before any search, confirm:

1. **Mode** (infer, confirm if ambiguous):
   - **Single-feature** — user described one specific feature. Default when in doubt.
   - **Roundup** — user asked for "this week's roundup", "what shipped this week", or described multiple unrelated features.
2. **Feature description(s)** (required):
   - Single-feature mode: a short paragraph on what the feature does and who it's for. Don't proceed without it — it's the filter that makes the PR search useful.
   - Roundup mode: either the user lists the features themselves, or you cluster candidate PRs into features in Step 4 and confirm the groupings with the user before drafting.
3. **Time window** (ask if not given). Accept natural phrasing ("past 2 weeks", "since April 1", "since the v2.3 tag"). If the user doesn't say, default to the **last 2 weeks** and note that you defaulted.
4. **Contributors to thank** (ask once, briefly). The user's announcements often include `@mentions` for specific contributors. Ask "anyone you want to @mention or thank?" — don't invent names from PR authors. If the user says no, skip the thank-you line entirely.

Optional:
- Explicit list of repos to include or exclude.

## Step 1 — Read the repo reference, then list org repos

First, read `references/m2m-repos.md` (relative to this SKILL.md) so you have baseline knowledge of each documented repo and the domain-to-repos lookup table.

Then list all non-archived repos in the org:

```bash
gh repo list Month2Month --limit 200 --no-archived \
  --json name,description,isArchived,pushedAt,visibility
```

This gives you the **actual current set** of repos — some won't be in the reference file (new or undocumented ones). For anything not in the reference, its `description` field from this query is your best signal of what it does.

## Step 2 — Shortlist repos likely to contain feature PRs

Using the feature description plus the domain-to-repos table from the reference file and the `gh repo list` descriptions, narrow down to repos plausibly in scope. Skip:
- Repos whose description / documented role is clearly unrelated (e.g. a design-tokens repo for a pricing feature).
- Repos that haven't been pushed to within the time window (`pushedAt` from Step 1).
- Archived repos (already excluded by `--no-archived`).

State the shortlist in one sentence before querying, e.g.:

> Based on the feature description (guest rescheduling), I'll search `holidale-web` (booking lifecycle), `m2m_pms_frontend` (internal UI), and `spark-ssr` (public UI). Skipping `financial_god`, `mia-m2m-mcp`, and the mobile repos since they don't look related. OK to proceed?

Wait for a nod unless the user already told you which repos to search.

**Err on the side of slightly wider.** A cross-cutting feature often quietly touches `holidale-web` for an API change even if the user only mentioned the UI. See the "Things to watch for" section of the reference file.

## Step 3 — Pull merged PRs per repo

Run per-repo PR searches in parallel — they're independent. Use `gh -R` to target each repo explicitly; you do not need to be `cd`'d into a clone.

```bash
gh -R Month2Month/<repo> pr list \
  --state merged \
  --search "merged:>=<YYYY-MM-DD>" \
  --limit 100 \
  --json number,title,url,body,mergedAt,author,labels
```

For date ranges, use `merged:<YYYY-MM-DD>..<YYYY-MM-DD>` syntax.

If a single repo returns more than ~50 PRs, do a cheap first pass on **titles only** — keep ones that plausibly touch the feature by keyword or theme, then fetch bodies only for survivors. Don't read 100 full PR bodies top-to-bottom.

If `gh` errors on auth, stop and ask the user to run `gh auth login`. Don't try to paper over auth issues.

### Resolving tags or SHAs to dates

If the user scopes by tag or commit SHA rather than a date, resolve to a date without needing a local clone:

```bash
gh api repos/Month2Month/<repo>/commits/<tag-or-sha> --jq '.commit.committer.date'
```

Then use that date's `YYYY-MM-DD` portion in `merged:>=...`.

## Step 4 — Filter / cluster PRs

### Single-feature mode

Read the title + body of each candidate and decide: does this PR contribute to the feature the user described? Keep it only when the connection is clear from the PR itself — a feature name in the title, an issue link to a relevant ticket, a body that describes a user-facing change matching the feature.

Be conservative. Cross-department readers get confused by tangential PRs ("flaky test fix", "dependency bump", "refactor of X service"), and including them dilutes the announcement. When in doubt, put the PR in a **"maybe" pile** and ask the user at the end rather than silently including it.

If zero PRs match, say so plainly. Don't pad the announcement with filler to cover the absence.

### Roundup mode

Cluster PRs into features rather than filtering against one. Useful grouping signals (in order of strength): shared Linear/Jira ticket ID across PRs, shared feature branch prefix, overlapping subject-area keywords in titles and bodies, clusters of PRs that landed close together across related repos. Ignore chore / dep-bump / flake-fix PRs unless they're large enough to be a feature in their own right ("Remove dead code: 21,000 lines" can be its own line item).

Before drafting, present the clustered feature list back to the user — "Here's what I'm seeing: [feature A], [feature B], [feature C]. Anything to add, drop, or rename?" — and wait for a confirmation. Roundups read poorly if you guess the groupings wrong.

## Step 5 — Draft the announcement

The audience is internal, cross-department. This is a Slack/internal-post-style announcement in the user's voice — not a press release and not a marketing email.

### Choose a mode

**Single-feature mode** (default): one feature, focused announcement. Use when the user described a single feature to announce.

**Roundup mode**: multiple distinct features across the company shipped in the same window. Use when the user asked for "a weekly roundup", "what shipped this week", or described more than one unrelated feature. Do not combine unrelated features into single-feature mode — it dilutes both.

If unsure, ask.

### Voice (both modes)

Match the user's actual voice — warm, matter-of-fact, direct. Not corporate, not marketing-y, not pumped-up. Key patterns:

- **Greeting**: `Hi team,` (always, on its own line).
- **Excitement framing**: "I'm excited to announce that X is now online!" for single features. "Quick roundup of what's shipped this week across our products:" for roundups. Excitement is about the feature going live, not about personal pride or team effort.
- **Plain language, no jargon.** No "middleware", "endpoint", "SDK", "migration", "MCP", "webhook", "refactor", "WebSocket", "RPC", "monolith". If you must reference a technical concept, translate it ("a background process", "the part of the site that…", "the PMS system").
- **No implementation-scope framing.** No "cross-repo push", "shipped across X services", "a big lift across our stack". Abstract implementation complexity away from the body entirely. Don't name repos in the body.
- **Lead with what's possible, not what was built.** "You can send prospects a link to..." — not "we built an API that...".
- **No self-praise, no team-praise closers.** No "genuinely proud of this one", "great work team", "huge win for the team". Keep closers functional: "Please give it a try and let me know if you have any questions or feedback!"
- **Thank specific contributors by @mention when the user tells you who contributed.** Don't invent contributor names from PR authors — ask the user if anyone should be credited.
- **Forward-looking line** near the end is characteristic: "The future vision for this tool is...", "the first of more pillar pages to come", "a foundation we'll keep building on". Include one when it fits.
- **No department callouts** ("For Sales:", "For CS:"). Don't assume team structure.
- **Sign-off**: `Thanks,` on its own line at the end (single-feature). For roundups, the closer line is often the sign-off and `Thanks,` is optional.

### Formatting (both modes)

- **No markdown headers (`#`, `##`).** Section titles are written as questions ("How to send a lease preview?") or short bolded phrases, on their own line, followed by the content.
- **Numbered lists for workflows** (how-to steps).
- **Bulleted lists for feature inventories** (e.g. "Current features:").
- **Inline URLs**, not hidden behind link text.
- **Image placeholders**: write `image.png` on its own line where a screenshot belongs, so the user knows to paste one in.
- **No subject line, no frontmatter.** No "Shipped: X — Y" / "For: Z" blocks. No engineering-reference PR appendix in the body (the PR list goes at the very bottom under a comment the user can delete before sending).

### Scaffold — Single-feature mode

```markdown
Hi team,

I'm excited to announce that <feature name> is now online!

<One paragraph in plain language describing what the feature does and what it unlocks. Lead with what the internal user (or external user) can now do, not how it was built.>

How to <primary action, e.g. "send a lease preview">?
1. <step>
2. <step>
3. ...

How does the <other party, e.g. "owner"> experience <feature>?
1. <step>
2. ...

Current features:
- <capability>
- <capability>
- ...

The future vision for this tool is <short, concrete forward-looking line — what this is building toward>.

Please give it a try and let me know if you have any questions or feedback!

Thanks,

<!-- PR reference (for your records — delete this block before sending):
- <repo>: #<PR>, #<PR>
- <repo>: #<PR>
-->
```

Omit "How does the other party..." if the feature has only one user type. Omit "The future vision..." if you don't have a concrete line — don't force it.

### Scaffold — Roundup mode

Group features by product area, with an emoji per area. Each feature gets a bolded name (no `##`), a short prose paragraph, an `image.png` placeholder where relevant, and a bullet list of capabilities. Major features can use a "Before / Now" framing and a "What this unlocks in practice:" subsection for downstream impact.

Typical area buckets and emojis (adapt to what actually shipped):

- 🌐 `month2month.com` Updates — public site
- 🛠️ Internal PMS Updates — internal admin tools
- 📱 Mobile — vendor / owner apps
- 🤖 AI & Automation — Mia, MCP, agent features
- 🏗️ Platform / Infrastructure — foundational improvements

```markdown
Hi team,

Quick roundup of what's shipped this week across our products:

🌐 month2month.com Updates

**<Feature name>**
<One-paragraph description of what shipped and why it matters. Can include an editorial beat or two about the strategic context — e.g. "a deliberate step forward in our SEO strategy".>
image.png
<Optional sub-header, e.g. "What visitors will see:">
- <capability>
- <capability>

**<Another feature>**
<Paragraph>
image.png
- <capability>
- ...

🛠️ Internal PMS Updates

**<Feature name>**
<Paragraph>
image.png
- <capability>
- ...

**<Major feature — e.g. TicketTaker upgrade>**
<Intro paragraph — what it is, refresher if needed.>
image.png
The journey so far:
Before: <where we were>
Now: <where we are>

What this unlocks in practice:
- <concrete outcome>
- <concrete outcome>

<Optional closing editorial line — e.g. "This is the harness engineering we've been building toward...">

Let me know if you have any feedback or want a walkthrough of anything here.

<!-- PR reference (for your records — delete before sending):
Per-feature:
- <feature>: <repo> #<PR>, <repo> #<PR>
-->
```

Only include area buckets that had something ship. Only include sub-headers ("What visitors will see:", "What this unlocks in practice:", "The journey so far:") when a feature is substantive enough to warrant them — most features just need a paragraph and a bullet list.

## Step 6 — Save and present

1. Slugify:
   - Single-feature: `release-notes-<feature-slug>-<YYYY-MM-DD>.md`
   - Roundup: `release-notes-roundup-<YYYY-MM-DD>.md`
2. Write the markdown to `/tmp/<filename>.md`.
3. In chat, print:
   - The full markdown block (so the user can copy-paste into Slack or email).
   - The file path.
   - A **"Confirm these?"** list of any "maybe" PRs from Step 4 that you didn't include, so the user can tell you to add them.
   - A one-line note if any defaults kicked in (time window, mode).

## Notes and gotchas

- **Org is `Month2Month`** (case-sensitive in some contexts). Always `gh -R Month2Month/<repo>`.
- **Search syntax:** `merged:>=2026-04-01`, `merged:2026-03-01..2026-04-01` for ranges. `is:merged` is implied by `--state merged` but can be layered on for complex searches.
- **Squash merges + bot authors:** `author.login` may be a bot or release manager, not the feature owner. Don't feature authors in the announcement unless the user asks.
- **Linear/Jira ticket IDs** in PR bodies are often a better cross-repo grouping signal than titles. If you see the same ticket ID across repos, those PRs almost certainly belong to the same feature.
- **Undocumented repos are normal.** If you find relevant PRs in a repo not listed in `references/m2m-repos.md`, include them in the PR list — just describe the change by what it does, not by what the repo is, and never name the repo in the announcement body.
