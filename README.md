# Agent skills

Skills for coding agents that take work from an agreed design to merged PRs (Claude Code format: a folder with a `SKILL.md` and optional `references/`). Copy a folder into `~/.claude/skills/` for every project, or into a repo's `.claude/skills/` for that repo only.

| Skill | What it does | Needs |
|---|---|---|
| `slice-work` | Turns an agreed design into vertical slices, one issue each: end to end, about 200 hand-written lines across at most 8 files, with the files and the test named, never one layer at a time. Shows the plan, creates the issues after the person's ok, hands the ids to `ship-wave`. | a tracker |
| `ship-wave` | Leads a batch of issues to merged PRs: one sub-agent per issue in its own worktree, draft PRs, pr-gate on each, at most 3 PRs waiting on the person at once, merge in dependency order, tracker updates, one digest for the person. | `pr-gate` |
| `pr-gate` | Verifies a PR and sends it down one of two lanes: the agent lane (the agent merges) or the person's lane (you merge, from 1-3 look-here points). Sends back a PR that is not one vertical slice; collects evidence and runs a fresh-context review (none for docs or tests only), plus a security review and `/code-review high` only on sensitive paths. The repo's `.claude/risk-paths.md` (or the default list in `references/`) sets the paths and the repo's lane setting: `lane: mvp` lets the agent merge everything except sensitive paths and one-way doors; `lane: live`, the default, only docs, tests and regenerated files. Reviews report blocking findings only, and agents file no issues. | `gh` |

**From [mattpocock/skills](https://github.com/mattpocock/skills)** (MIT, copied unchanged, see [CREDITS.md](CREDITS.md)). Run `/setup-matt-pocock-skills` once per repo first: it records the tracker and labels the others use.

| Skill | Use it for |
|---|---|
| `grill-me`, `grill-with-docs` | Interviewing you about a plan until it's settled (both call `grilling`); `grill-with-docs` also writes ADRs and a glossary (`domain-modeling`). |
| `to-tickets` | Cutting a plan into vertical-slice tickets with blocking edges. |
| `prototype` | A throwaway prototype to settle a UI or state question. |
| `improve-codebase-architecture` | Finding the worst tangles in a messy codebase (`codebase-design`). |
| `wayfinder` | Work bigger than one session, as a map of decision tickets (calls `research` for facts outside the repo). |

The `guardrails` skill (onboarding an existing repo) lives with its CLI in [agent-guardrails](https://github.com/freshmick/agent-guardrails).

To upload to claude.ai, drop the `disable-model-invocation` line from Matt's `SKILL.md` files first: claude.ai accepts only `name`, `description`, `license`, `compatibility`, `metadata` and `allowed-tools`.

## When to use which

```mermaid
flowchart TD
    A[Design conversation with Claude<br/>what to build, architecture, technologies<br/>optional: /grill-me from mattpocock-skills] --> B["/slice-work"]
    B -->|questions| A
    B -->|plan table, your ok| C[One issue per vertical slice<br/>criterion, test, files, blocking links]
    C --> D["/ship-wave &lt;ids&gt;"]
    D --> E[Sub-agent per issue<br/>own worktree, self-review, size check, draft PR]
    E -->|over the cap| B
    E --> F["/pr-gate (run by ship-wave)"]
    F -->|not one slice| B
    F -->|blocking findings, max 3 rounds| E
    F -->|agent lane| G[Agent merges]
    F -->|your lane| H[You: read the gate comment<br/>decide the 1-3 look-here points, merge]
    G --> I[Wave digest]
    H --> I
    I -->|left out, questions, recurring findings as a new check first, a CLAUDE.md line only when no check can express it| A
```

- **Design** is a conversation, not a skill here. End it when you can say what the user can do afterwards and how a test would prove it.
- **`/slice-work`** when the design is agreed and implementation is next. Run it again when ship-wave or pr-gate sends an issue back as too big or not a slice.
- **`/ship-wave`** with the issue ids slice-work printed. It runs pr-gate itself; you don't call pr-gate during a wave.
- **`/pr-gate`** on its own only for a PR that didn't come from a wave: a hand-written PR, or one from another tool.
- **You** read the digest and the gate comments of PRs in your lane. Not the diffs: the gate's look-here points are the decisions, and the evidence section says what was run and seen.

To update a skill on claude.ai: zip the skill folder (folder at the zip root, no `__pycache__`), then Customize > Skills > the skill > More options > Replace.
