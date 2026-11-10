# CLAUDE.md

This repo is a Claude Code skill (`SKILL.md` + `reference.md`). It was written with Claude and is maintained with Claude.

## Working on the skill

- `SKILL.md` body stays under ~150 lines. It loads into context every time the skill triggers, so every line must change agent behaviour or be cut. Move optional detail to `reference.md`.
- Write in the imperative and keep it concrete. Use kitchen terms only as the metaphor's labels: no puns, no motivational text.
- Don't claim a capability (model names, Agent tool parameters, effort levels) unless it was confirmed in a real Claude Code session. If unsure, say so in the skill.
- Keep the frontmatter `description` focused on triggers. It decides when the skill loads.

## Versioning

- Each version is a proof of concept (POC) run against real feature work. Findings go in `results/` (gitignored, never commit it).
- To make a new version: edit the skill, add a row to the version table in `README.md`, commit, then tag it (`git tag vN`) and push the tag.
- Base each change on evidence in `results/`, not on taste. Name in the commit body which finding drove the change.

## Commits

- Write short commit messages in imperative form.
- Don't add `Co-Authored-By` trailers or "Generated with" lines, and never mention people's names.
