# stranger-skill

A Claude Code skill that encodes the **build → review → test** loop I actually use day-to-day — distilled from shipping DotMD (a Markdown-native productivity suite: docs, slides, sheets, native shells, AI everywhere). It is opinionated, repeatable, and biased toward shipping working code without regressions.

## What's inside

- `skills/stranger-skill/SKILL.md` — the skill itself, organized as three phases (Build, Review, Test) with concrete steps and the rules that prevent the failure modes I've actually hit.

## Installing

Drop the `skills/stranger-skill/` directory into your Claude Code skills path (`~/.claude/skills/` or the project-local `.claude/skills/`). Invoke from a chat with `/stranger-skill` or let Claude auto-invoke when the task description matches.

## Why this exists

Most "best practice" guides describe what *should* happen. This one captures what actually happens when a single dev ships a multi-surface app, reviews their own PRs, and runs the full test pipeline. Every rule here exists because it prevented (or was learned from) a real outage.

## License

MIT

## Used by DotMD for live GitHub testing

This repo is **intentionally minimal and public** so it can serve as a stable target for a fine-grained PAT in DotMD's e2e + manual test flows (Settings -> GitHub -> live repo + branch dropdowns -> Sync).

If you're a DotMD agent, see `AGENTS.md -> GitHub Settings test PAT` in the marc-prod monorepo for the create recipe. The PAT lives ONLY in `dotmd/apps/web/.env.local` as `GITHUB_PAT_STRANGER_SKILL=...` (gitignored). It is scoped to this repo only (Contents R/W, Metadata R) so the blast radius of a leak is contained to a public docs repo with no secrets.