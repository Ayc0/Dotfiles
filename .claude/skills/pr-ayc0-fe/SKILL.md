---
name: pr-ayc0-fe
description: Create or update PR titles and descriptions following Ayc0's personal frontend PR style. Use when the user asks to create a PR, write a PR description, update a PR description, or mentions PR title/description work.
user-invocable: true
---

# PR Description — Ayc0's frontend style

Create or update pull request titles and descriptions matching Ayc0's hand-written style — distinct from AI-generated PRs.

## Usage

```
/pr-ayc0-fe                       — create title + description for current branch
/pr-ayc0-fe update                — re-analyze full diff and rewrite for current branch's PR
/pr-ayc0-fe update <PR number>    — update a specific PR
```

> **"Update" means re-analyze the entire PR diff from base to HEAD** — rewrite the description from scratch based on the complete state of the branch, not just the latest commits.

## Global CLI rule

Every `gh pr` command MUST include `--repo <org>/<repo>`. No exceptions.

## Step 1 — Detect existing PR

```bash
git branch --show-current

PR_JSON=$(gh pr view --repo <org>/<repo> --json number,title,body,state,isDraft,baseRefName,headRefName 2>&1)
GH_EXIT=$?

if [ $GH_EXIT -ne 0 ]; then
  if echo "$PR_JSON" | grep -q "no pull requests found\|Could not resolve to a PullRequest"; then
    PR_EXISTS=false
  else
    echo "gh error: $PR_JSON" >&2
    exit 1
  fi
else
  PR_EXISTS=true
fi
```

## Step 2 — Resolve the base branch

- If `PR_EXISTS=true`: `BASE=$(echo "$PR_JSON" | jq -r '.baseRefName')`
- For a specific PR number: `BASE=$(gh pr view --repo <org>/<repo> <number> --json baseRefName --jq '.baseRefName')`
- If `PR_EXISTS=false` (new PR): detect the repo's default base branch — never hardcode `main` or `master`, repos use different conventions:

  ```bash
  BASE=$(git symbolic-ref refs/remotes/origin/HEAD --short 2>/dev/null | sed -e 's@^origin/@@')
  ```

  If that fails, or stacked branches are suspected (the current branch should target another feature branch, not the default), ask the user.

Never leave `<base-branch>` as a literal placeholder. If unresolved, stop and ask.

## Step 3 — If creating a new PR

1. **Branch prefix**: if the repo has a configured branch prefix convention, the branch must use it. If missing, rename with `git branch -m <prefix>/<rest>` after warning the user if the branch is already pushed.
2. **Push if needed**: check `git rev-parse --abbrev-ref --symbolic-full-name @{u}` — push with `-u origin <branch>` if no upstream.
3. **Always create as draft** — use `--draft`.
4. Generate title + body using this skill's guidelines and run directly (see "Approval workflow" — only stop if you have a blocking question):

```bash
gh pr create \
  --repo <org>/<repo> \
  --base "$BASE" \
  --draft \
  --title "<title>" \
  --body "$(cat <<'EOF'
<body>
EOF
)"
```

## Step 4 — If updating an existing PR

**Do not change draft state.** Only apply `--draft` on creation. Never call `gh pr ready` unless asked.

### Extract & lock all media first

Before rewriting, scan the existing description and extract verbatim:

- Markdown images `![](...)` and HTML `<img>` tags
- `<video>` blocks, video service links (Loom, etc.), `.mp4`/`.gif`/`.webm` URLs
- GitHub user-attachment URLs (`github.com/user-attachments/...`, `user-images.githubusercontent.com/...`)
- Figma / design tool links
- Any external link added by hand

For each, note the section and position. Re-inject every item back into the same section after rewriting. Never drop, reformat, or resize.

### Diff to analyze

```bash
# current branch:
git diff "$BASE"...HEAD

# specific PR:
PR_HEAD=$(gh pr view --repo <org>/<repo> <number> --json headRefName --jq '.headRefName')
git diff "$BASE"..."$PR_HEAD"
```

Rewrite based on the **full diff**, not just new commits. Update the title too if scope shifted.

### Apply

```bash
gh pr edit --repo <org>/<repo> [<number>] \
  --title "<title>" \
  --body "$(cat <<'EOF'
<body>
EOF
)"
```

## Approval workflow

**Don't ask for approval before running `gh pr create` / `gh pr edit`.** Generate the title + description and run the command directly.

**Only stop to ask the user** when you have a genuine open question that blocks generating a sensible PR — e.g.:

- Ticket link / motivation context is missing and can't be inferred from the branch or commits.
- The diff spans multiple unrelated scopes and you can't pick a coherent title.
- Required media (Before/After screenshots, preview URL details) is needed and not derivable.
- The base branch is ambiguous (suspected branch stacking, no PR yet, no clear convention).

If you ask a question, wait for the answer, then proceed without further confirmation. After running the command, show the final title + description and the PR URL so the user can review.

## Title

### Format

```
[<scope>] <description>
[<scope>][<sub-scope>] <description>
```

### Scope rules

**The scope names a thing in the codebase, and matches that thing's natural casing.** Don't reach for predefined categories — write the identifier the way it appears in the code or filesystem.

- **Component** (e.g. a React component) → PascalCase: `[Popover]`, `[SearchBar]`.
- **Function** → camelCase: `[useDebounce]`, `[formatDate]`.
- **Module / folder / package** → kebab-case: `[design-system]`, `[rich-link]`, `[page-login]`.
- **Plain words / concepts / domains** → lowercase: `[auth]`, `[search]`, `[unreachable]`.
- **Files** → keep their actual name: `[CLAUDE.md]`, `[README]`.

Sub-scopes follow the same rule for whatever they name. Combine freely: `[<module>][<Component>]`, `[<module>][<Component>][<feature>]`, `[<Component>][<word>]`.

### Description rules

- **Lowercase first letter** (imperative): "add", "remove", "fix", "update", "move", "split", "create", "use", "drop", "make", "show", "migrate", "re-implement", "scaffold", "render", "extract".
- **No trailing period.**
- **Terse for small fixes** (`fix TS`, `fix preview`, `apply feedback`), **descriptive for features/refactors** (`scaffold V2 overview with TOC and 4 sections`).
- **Code identifiers as-is** — backticks fine in title: ``add `shouldFollowContent` prop``, `` migrate confirmation modals to `openConfirmationModal` ``.
- **Abbreviations OK**: TS, FF, UI, API, FE, TOC, AI, VRT.
- **Use `&` to combine** related changes: `fix TS & export variables to utils & types`.
- **No emojis. English. No ticket ID in title** (mention in Motivation instead).

## Description

### Template

```markdown
## Motivation

<Ticket link first, if any>

<Optional: design link / screenshot / context paragraph>

<1-3 sentences explaining the why. For larger work, a short narrative is fine.>

## Changes

- <bullet 1, backtick all code identifiers>
- <bullet 2>
  - <nested detail if needed>
- <bullet 3>

## QA Instructions

<Optional FF gating note: "Behind FF `flag-name`. Toggle with <how to toggle>.">

<Preview env URL, if the repo has one>

| Before               | After                |
| -------------------- | -------------------- |
| `<screenshot/video>` | `<screenshot/video>` |

## Blast Radius

<Short paragraph or bullets. Mention FF gate.>
```

### Section-by-section guidance

#### Motivation

- **Lead with the ticket link** on its own line if there is one. Just the URL alone is fine.
- Optionally follow with a design screenshot, design tool link, or a chat/PR reference. Those are pointers — keep them, no prose around them.
- **Default to ~2 short sentences.** State the problem and the fix the way you'd tell a teammate at the watercooler: _"X was confusing because Y, so I made it Z."_ That's the whole section for most PRs.
- **Don't walk through worst-case scenarios.** Don't editorialize ("...the viewer has no way to mentally line up the three"). The reader will figure that out from the screenshots and the Changes bullets.
- **Incomplete trailing sentences are fine** when the meaning is clear. Terseness > polish.
- For substantial architectural work, a longer **personal narrative is OK and matches the user's voice** — e.g. "Everything started with this comment: <link>… and this stuck with me for 2 reasons…". Don't force this; only when there is a genuine backstory and the PR is large enough to warrant it.
- For tiny PRs, Motivation can be a single sentence like `Share the same logic of cleaning the text between emails & page display`.

#### Changes

- **Stay terse and product-focused.** Each bullet should be a product/user-visible decision or a non-obvious architectural choice. A reviewer reads the Changes section to know _what shifted_, not to relive the diff. **2-4 top-level bullets is the sweet spot** — if you wrote 7, you're probably listing wiring.
- **Drop diff recaps.** Things visible from reading the code itself don't belong: loading / error / empty state mechanics, component swaps a reviewer will see, internal refactors with no behavior change (making a prop required, inlining a value, moving a hook call locally vs prop-drilling). If removing the bullet would mislead someone, keep it; if removing it would only deprive them of trivia, drop it.
- **Drop per-caller wiring breakdowns.** Lines like _"`PanelA` passes X, `PanelB` renames Y, the other callers only pass Z"_ belong in the diff, not the description. If a reviewer needs to know which callers were touched, they'll grep.
- **Drop edge-case rules of behavior.** Lines like _"the tooltip shows as soon as any of the displayed values differ — so the `preview === original ≠ user` case is now covered"_ live in tests, not Changes.
- **Bulleted list**, nesting allowed (2-3 levels max).
- **Backtick all code identifiers**: components, hooks, FF names, file paths, types, URL params.
- **First-person casual is fine** ("I'm fetching", "I'm rewriting", "I'm checking the types") — matches the user's voice. Don't fabricate first-person; only when it reads naturally.
- **Bold key flow/section names** (e.g. `**<flow name>** flow:`) when introducing multiple flows or grouped concepts.
- Use `→`, `:`, and `(aka …)` for transformations and aliases.
- **Cross-reference PRs** with `#NNNNNN` or full URL, and **colleagues with @handle**.
- **Don't list every file modified** — describe meaningful changes. File path mentions in backticks are fine for important moves/renames.
- For asides, casual interjections match the voice ("Some weird decisions were made I know but, hear me out:", "To do so, I copied some logic from previous folders (but mentioned it) to avoid changing runtime code.").

#### QA Instructions

- **Almost always include a `Before | After` markdown table** with screenshots or videos. Use multi-row tables for multiple cases, optionally with a third column for the URL link.
- **Use subsections** (`### <case name>`) when there are multiple distinct cases to QA.
- **Include the preview environment URL** when applicable. If the repo's preview URLs are derived from the branch name (e.g. a hash of it), compute it from `git rev-parse --abbrev-ref HEAD` rather than leaving a placeholder.
- **Mention the FF flag and how to toggle it** when the feature is gated.
- For pure refactors with no visual change: `Only logs` / `No visual change, behavior is unchanged.` is fine.
- **Preserve any user-pasted screenshots/videos** verbatim when updating (see media extraction in Step 4).

#### Blast Radius

- Keep it short. Range from 1 word (a domain name) to a short bullet list.
- **Always mention the FF gate** if there is one: "Gated by the `<flag-name>` FF — no impact outside opted-in orgs."
- For lib/shared changes, call out affected consumers ("All popovers. But this is a new prop to enable so should be fine").
- For prod-impact changes, be explicit: e.g. "**PROD** for <area>".

### Trimming a verbose first draft

When an initial draft is too long, these are the categories of content that almost always get cut vs kept. Use this as a calibration target — not a checklist, but a sense of *what kinds* of content survive the trim.

**What got cut and why:**

- Worst-case scenario walkthroughs in `## Motivation` → the diff + screenshots make them redundant.
- Sentences in `## Motivation` that restate what's already in `## Changes` (or vice versa) → say it once.
- Per-caller wiring breakdowns in `## Changes` (`PanelA passes X, PanelB renames Y, the other callers only pass Z`) → diff detail, not a product decision.
- Edge-case behaviour rules in `## Changes` (`shown when A differs from B and C differs from D`) → test concern, not reviewer concern.
- Notes about diff-visible refactors (a function moved back to being called internally, a prop renamed, a helper inlined) → reading the diff is faster than reading the prose.

**What was kept:**

- Ticket / TODO / cross-PR references (pointers, not prose).
- 1-2 sentences naming the problem and the fix in plain language ("X was confusing because Y, so I'm doing Z").
- 2-4 bullets stating the product-visible shape of the change.

Rule of thumb: if your draft is more than ~3× the length you'd need to describe the PR to a teammate verbally, cut.

### Voice

- Personal, casual, direct. First-person OK ("I'm fetching", "I switched from X to Y").
- Asides and rationale-as-narrative are welcome on substantial PRs ("hear me out", "Some weird decisions were made I know but…").
- **Don't sound like AI-generated PRs**: avoid generic phrases like "This PR introduces…", "This refactoring maintains identical user-facing functionality while simplifying internal implementation", "The changes are isolated to…".
- **No machine-generated metadata comments** (e.g. `<!-- *-meta … -->` markers tied to specific PR-generation tools).
- **No "PR by <tool> — View session in <service>" footer.**

## Core philosophy

- Tell the reviewer **why** the PR exists, **what** it does, and **how to verify** it.
- Backtick everything that is code so the description scans.
- A Before/After visual is worth a thousand bullets — include them whenever there's a UI change.
- Don't pad. Terse for small PRs, narrative for big architectural ones.

## Anti-patterns (what to avoid)

- Listing every file changed.
- Explaining the diff line-by-line.
- Padding `## Changes` with implementation details visible in the diff — loading/error swaps, internal refactors, prop signature tweaks. Those aren't product decisions.
- Per-caller wiring breakdowns in `## Changes` (`PanelA passes X, PanelB renames Y...`). The diff is the source of truth for wiring.
- Edge-case behavior rules in `## Changes` (`tooltip shows when A ≠ B and C ≠ D`). Tests own those.
- Walking through worst-case scenarios in `## Motivation`. Show the screenshot instead.
- Editorializing in `## Motivation` ("...the viewer has no way to mentally line up the three"). State the problem and stop.
- Generic AI-flavored summaries ("This PR introduces a new feature that…").
- Dropping the ticket link from Motivation when one exists.
- Omitting Before/After when there's a visible UI change.
- Including machine-generated metadata HTML comments or "PR by <tool>" footers.
- Putting the ticket ID in the title.
