# Risk paths (read by the pr-gate skill)

lane: mvp

This repo has no users, so the agent merges every PR except one that touches a sensitive path or is marked `Door: one-way` (Mick, 12 November 2026). Skill files change what agents do, so a PR that edits `skills/**` is never docs-or-tests only: it is ordinary and gets one fresh-context reviewer.

One glob per bullet, in backticks, gitignore style. A file matching a sensitive glob is sensitive even if it also matches a docs-and-tests glob. The gate may move a PR to the person's lane, never to the agent lane.

## sensitive
- `.claude/**` - the rules agents follow, including this file
- `CLAUDE.md`
- `AGENTS.md`
- `.github/**`

## docs and tests
- `README.md`
- `CREDITS.md`
- `licenses/**`
