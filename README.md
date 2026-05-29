# shared-skills

Shared [Claude Code](https://claude.com/claude-code) skills, kept in
`.claude/skills/`. Each skill is a directory with a `SKILL.md` (plus any assets).

## Skills

<!-- SKILLS:START -->
| Skill | What it does |
|---|---|
| `/pm-define-tasks` | Turn a rough work idea into one or more well-specified tasks through targeted questions — never inventing details. |
| `/rg:pr-dossier` | Generate a self-contained HTML "PR reference dossier" — a scannable review brief that explains what changed, how risky it is, where to look first, and how to check it — and save it under docs/pr-reference/ so it travels with the PR. |
<!-- SKILLS:END -->

## Adding a skill

1. Create `.claude/skills/<name>/SKILL.md` with `name` and `description`
   frontmatter (add an `assets/` dir if the skill needs files).
2. Run `bin/sync-readme` to refresh the table above.

The table between the `SKILLS:START` / `SKILLS:END` markers is generated from
each skill's frontmatter — don't edit it by hand. A `Stop` hook in
`.claude/settings.json` runs `bin/sync-readme` at the end of every Claude Code
turn, so the table stays current automatically. `bin/sync-readme --check`
fails if it has drifted (useful in CI).
