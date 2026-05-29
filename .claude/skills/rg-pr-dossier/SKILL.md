---
name: rg:pr-dossier
description: >-
  Generate a self-contained HTML "PR reference dossier" — a scannable review
  brief that explains what changed, how risky it is, where to look first, and
  how to check it — and save it under docs/pr-reference/ so it travels with the
  PR. Built from the branch diff, commits, and (if present) a matching plan file
  with no prompts, so it can run unattended from a PR-finishing command. The
  file has no external dependencies and opens in any browser; a reviewer can
  also drag-drop it into the PR conversation to get a rendered GitHub
  attachment. Stack-agnostic: it detects the project's test/lint commands rather
  than assuming a framework. Use when finishing a feature, opening a PR, or when
  you want a reviewer-facing change summary.
user-invocable: true
argument-hint: "[optional: PR title or context] [--output PATH] [--base BRANCH]"
allowed-tools: ["Read", "Grep", "Glob", "Bash", "Write", "Edit"]
---

# PR Reference Dossier

Produce **one self-contained `.html` file** that helps a reviewer start fast: what
changed, how risky it is, where to look first, what to skim, and how to check it.
The file is filled from facts already in hand (branch diff, commits, and a plan
file if one exists) — **never prompt the user**, so this is safe to run unattended
from a PR-finishing command. It also works standalone.

The artifact lives in the repo at `docs/pr-reference/<branch>.html`. Because it is
committed on the feature branch, it shows up in the PR's **Files** tab, the PR body
can link to it, and a finishing command can post a PR **comment** linking it
(conversation tab) via the comments API. There is no API to upload a file
*attachment* (the drag-drop endpoint is web-UI only), and an uploaded `.html`
downloads rather than renders anyway — so the committed-blob link is the
best-supported, private-safe delivery. This skill just produces the file and tells
the caller where it is.

## Step 0: Resolve inputs (no prompts)

```bash
branch=$(git branch --show-current)

# Base branch to diff against (override with --base)
base=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
[ -z "$base" ] && base=$(git rev-parse --verify origin/main >/dev/null 2>&1 && echo "main" || echo "master")

# Output path (override with --output). Default: docs/pr-reference/<branch>.html
out="docs/pr-reference/${branch}.html"
mkdir -p "$(dirname "$out")"
```

- `--output PATH` overrides the destination.
- `--base BRANCH` overrides the diff base (default: the repo's default branch).
- Any other free text is treated as a **PR title / context hint** for the header
  and summary.

If `out` already exists (a re-run), overwrite it — the dossier should reflect the
current diff.

## Step 1: Gather context from the diff (the only source of truth)

Run these and read the results. **Do not invent anything the diff doesn't show.**

```bash
git log "$base"..HEAD --pretty=format:"%h %s"      # commits on this branch
git diff "$base"..HEAD --stat                       # files changed + churn
git diff "$base"..HEAD --name-only                  # bare file list for risk scan
```

Find a matching **plan / spec file** if the repo keeps one: try `plans/<branch>.md`,
`docs/plans/<branch>.md`, or scan `plans/`, `docs/plans/`, or `docs/` for a stem
matching the branch. If found, read its frontmatter (e.g. `title`, issue id) and any
`## Goal`, `## Decisions`, `## Open Questions`, and `## Scope Boundaries` (or
similarly named) sections. The plan is the best source for *why this approach*,
*tradeoffs*, and *known gaps* — sections you otherwise cannot fill honestly. If
there is no plan file, that's fine: leave those detail placeholders blank rather
than guessing.

Metadata for the header:

```bash
git config user.name                                # author
git rev-parse --short HEAD                           # commit sha
gh repo view --json nameWithOwner -q .nameWithOwner 2>/dev/null  # org/repo (best-effort)
```

Use today's date (the harness provides it) for the `Date` cell and footer.

## Step 2: Copy the template

The template lives at `assets/template.html` relative to this skill
(`.claude/skills/rg-pr-dossier/assets/template.html`). Copy it to `$out` and fill
that copy. **Do not rebuild the layout** unless the user explicitly asks for a
different format.

```bash
cp .claude/skills/rg-pr-dossier/assets/template.html "$out"
```

Then edit `$out` in place to fill the placeholders.

## Step 3: Fill the visible review path first

These must be useful even if the reviewer never expands a detail:

- **Header** — PR title (from the plan `title`, the context hint, or synthesized
  from commits), repo, branch, author, date, short SHA. PR `#` and reviewers stay
  as `[ ... ]` placeholders if unknown — leave the `fill` class on them.
- **Summary** — one sentence: what changed, who it affects, why it matters for
  review. Set its source badge `data-state` to `ai` (a drafted summary).
- **Quick numbers** — files changed (count from `--stat`), Tests (the project's
  test command — see Step 5, or `not run`), Data/schema (`migration or schema
  change` if migration/schema files changed, else `none`), Dependencies (manifest
  or lockfile touched → name it, else `none`).
- **Review guide** — "Look closely" = the genuinely risky files from the diff
  (auth/sessions/credentials, money/billing logic, schema/migrations, data
  transforms, external API calls), each with one phrase on what to check. "Okay to
  skim" = generated files, fixtures, renames, formatting-only churn.
- **What changed** — 2–5 observable behavior changes a reviewer can see, drawn
  from commit subjects and the diff.

## Step 4: Set risk flags honestly

Add the `active` class to any flag that applies, e.g.:

```html
<button type="button" class="flag active" data-sev="high">Auth or permissions</button>
```

The banner level is **computed** from active flags — never hand-set it. Be
conservative — when a change plausibly touches a sensitive area, flag it. If the
codebase handles a regulated or high-stakes domain (health, finance, personal
data), err further toward flagging. Map diff signals to flags:

| Signal in the diff | Flag to activate |
|---|---|
| Personal / regulated data — user records, PII/PHI, financial or health data | **Sensitive or personal data** (critical) |
| Payment, invoice, charge, billing-amount, or money-math logic | **Payments or billing** (critical) |
| Auth, sessions, passwords, OAuth, current-user, roles, permissions | **Auth or permissions** (high) |
| Credentials, ENV, tokens, keys, secrets | **Secrets or tokens** (high) |
| Changed response shape, serializers, public endpoint or exported API | **API behavior changes** (high) |
| Migration files, schema definitions, data backfills | **Database change** (medium) |
| New/changed external HTTP client, webhook, third-party SDK | **External service** (medium) |
| Background jobs, queues, async/worker code | **Background work** (medium) |
| N+1 risk, large loops, hot paths, new indexes for perf | **Could affect speed** (medium) |
| Dependency manifest or lockfile changes | **New or updated package** (low) |
| Config, deploy, CI workflow, infra changes | **Config or deploy setup** (low) |

## Step 5: Make the checks runnable

Fill **How to check it** with this project's *real* commands (not generic
placeholders). Detect them rather than assuming a stack — look, in order, for:

- A `bin/` wrapper directory (e.g. `bin/test`, `bin/lint`, `bin/dev`).
- `package.json` `scripts` (e.g. `npm test`, `npm run lint`, `npm run dev`).
- A `Makefile` / `Justfile` (`make test`, `just check`).
- A `Rakefile` / `Gemfile` (Ruby: `bundle exec rspec`, `bin/rubocop`).
- `pyproject.toml` / `tox.ini` / `pytest.ini` (Python: `pytest`, `ruff check`).
- `go.mod` (`go test ./...`, `go vet`), `Cargo.toml` (`cargo test`, `cargo clippy`).
- The repo's CI config (`.github/workflows/*`) — it lists the canonical commands.
- A `CONTRIBUTING.md` / README "Development" or "Testing" section.

Fill three lines with the closest match for this repo:

```
<focused test command for the changed area>
<lint / type-check / security-scan command>
<command to boot or exercise the app, for the manual check>
```

Include a security-scan command (e.g. `bin/brakeman`, `npm audit`, `bandit`) when
auth or sensitive code changed and the repo has one. Add a one-line **manual
check**: the exact steps to see the new behavior in the running app. If the plan
named a rollback concern, fill the **Rollback plan**; otherwise leave its
placeholder.

## Step 6: Fill the expandable details from the plan (don't guess)

- **Why this approach / Other options / Tradeoff / Links** — fill from the plan's
  `## Goal` and `## Decisions`. If there is no plan or it has none, **leave the
  placeholders** and keep the `fill` class. Do not invent a rationale. Set the
  source badge to `human` only for text taken verbatim from the plan; `ai` for
  drafted prose; leave `review` on anything that needs the author's eyes.
- **Design / flow change** — only if the diff shows a structural shift (new
  module, service, job, or changed call path). Otherwise leave the sketch
  placeholder.
- **Questions / gaps** — fill from the plan's `## Open Questions` and
  `## Scope Boundaries` (deferred work, intentional non-goals). These are exactly
  what a reviewer should not assume is settled.

## Step 7: Keep it self-contained, then report

- **No external resources** — no CDN links, web fonts, remote images, or external
  scripts. It must work from a downloaded attachment. (The template is already
  self-contained; just don't add any.)
- Leave a **visible placeholder** (keep the `fill` class) for anything you can't
  fill honestly — never a confident-sounding guess. The template's "Highlight
  blanks" button surfaces them.

Report the path and a one-line hint:

```
Dossier written: docs/pr-reference/<branch>.html
  • Open it in a browser for the interactive version.
  • When run from a PR-finishing command, it can be committed (Files tab),
    linked in the PR body, and posted as a PR comment (conversation tab) —
    all private-safe.
```

This skill **writes the file only** — it does not `git add`/commit and does not
open or modify the PR. A calling finishing command owns that ceremony: staging,
committing, linking the file in the PR body, and posting it as a PR comment via the
comments API.

## What this skill does NOT do

- Does NOT prompt — everything is derived from the diff, commits, and any plan.
- Does NOT invent rationale, tradeoffs, or rollback plans absent from the plan/PR.
- Does NOT assume a framework — it detects the project's real test/lint commands.
- Does NOT upload the file to GitHub as a rendered attachment — that endpoint is
  web-UI only (no API); a human does the 5-second drag-drop.
- Does NOT commit or push — the caller owns git.
