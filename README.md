# Agent skills

Skills for coding agents that take work from an agreed design to merged PRs (Claude Code format: a folder with a `SKILL.md` and optional `references/`). Copy a folder into `~/.claude/skills/` for every project, or into a repo's `.claude/skills/` for that repo only.

| Skill | What it does | Needs |
|---|---|---|
| `slice-work` | Turns an agreed design into vertical slices, one issue each: end to end, about 300 lines outside generated files, never one layer at a time. Shows the plan, creates the issues after the person's ok, hands the ids to `ship-wave`. | a tracker |
| `ship-wave` | Leads a batch of issues to merged PRs: one sub-agent per issue in its own worktree, draft PRs, pr-gate on each, merge in dependency order, tracker updates, one digest for the person. | `pr-gate` |
| `pr-gate` | Verifies a PR and gives it a risk tier. Sends back a PR that is not one vertical slice; collects evidence, runs a fresh-context review and a security review, sets the floor from the repo's `.claude/risk-paths.md` (or the default list in `references/`), labels the PR, posts one gate comment and merges only low-tier PRs itself. Out-of-scope review findings become follow-up issues, not commits. | `gh` |

To update a skill on claude.ai: zip the skill folder (folder at the zip root, no `__pycache__`), then Customize > Skills > the skill > More options > Replace.
