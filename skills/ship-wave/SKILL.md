---
name: "ship-wave"
description: Lead a wave of issues to merged PRs in any repo(s) with any tracker (Linear, GitHub Issues, Jira, or a plain list). Parallel sub-agents implement as draft PRs, pr-gate reviews and tiers each one, PRs merge in dependency order, and the tracker stays current. Use when the user says "/ship-wave", passes a handoff prompt, or asks to implement/kick off a set of issues.
argument-hint: "[issue IDs, project/milestone, or path to a handoff prompt]"
---

# Ship a wave

You are the lead engineer. Issues are the spec; this skill is the rules. Input: `$ARGUMENTS` (issue IDs, a project/milestone, or a handoff prompt path). A handoff prompt overrides this skill where they conflict.

**Engineering skills.** Steps below name skills from the engineering plugin (`testing-strategy`, `debug`, `deploy-checklist`, `architecture`). Review skills are not loaded here: pr-gate owns review. Load each with the Skill tool when its step starts, and tell sub-agents which one to load. They supply the lens; this skill and the repo's rules still decide scope, format and where output goes. If one isn't installed, the short checklist given in that step is the fallback.

## 0. Setup
Settle these from, in priority order: the handoff prompt, a matching `profiles/*.md` next to this file, `.claude/ship-wave.md`, the repos' AGENTS.md/CLAUDE.md/CI config. Ask the user only for what's left.
- Tracker (Linear, GitHub Issues, Jira, none) · repos and code host · worktree root (default `<repo>/../worktrees/`) · checks CI runs · merge convention and what merges trigger · follow-up rules · repo-specific rules.
- Identity: if `AGENT_GITHUB_LOGIN` is set (the agents' machine user), confirm `gh api user -q .login` matches it and that commits will carry it. If it is not set, say plainly in the plan message that this wave runs as the user's own GitHub account, then continue.

With no profile and repeat waves likely, offer to save one.

## 1. Before starting
- Read the instruction files of every touched repo, and every issue in full, including comments and blocking links. Don't re-decide what they settle.
- `git fetch` all repos. Never edit main checkouts.
- Put ambiguities (especially product behaviour) in one numbered md file with a/b/c options and a recommendation; give the user its path. For technical design choices, write each option the way `architecture` does (context, options with pros/cons, trade-off, consequences), kept to a few lines each. Don't block issues that don't depend on the answers.

## 2. Plan
- Order by blocking links and code dependencies; parallelize the rest, but serialize issues likely to touch the same files (shared styles, layouts, config, lockfiles).
- Batch same-class low-tier issues (docs, tests, wording) into one PR per wave. Everything else stays one issue, one PR. Split an issue only when it mixes a contract or sensitive change with ordinary code: that part gets its own PR. Before opening any PR, `gh pr list --search "<issue id>"` for one that already exists.
- Run pr-gate's floor script over the files each issue will likely touch (`python3 <pr-gate dir>/references/floor.py --risk-paths .claude/risk-paths.md --files <planned paths>`, no diff). If an issue floors at high or needs an ADR, write a one-page proposal for it (plan-mode output is the format: what changes, which contract, architecture or sensitive paths, the options and a recommendation) and wait for the user's approval of that issue. Start the others right away.
- Tell the user the plan in a few lines, including which issues wait for a proposal.

## 3. Implement (one sub-agent per issue)
- One issue, one fresh worktree from the updated default branch, one small PR opened as a draft (`gh pr create --draft`). Branch: the tracker's suggested name, else `<issue-id>-<short-slug>`.
- Only the acceptance criteria; anything else becomes a follow-up (step 6).
- Follow the repo's conventions and instructions (pass the step 0 rules to each sub-agent). Code: SOLID, KISS, DRY, low coupling and high cohesion, small single-purpose functions, dependency injection, clear names, config from env, no secrets committed. Add/update tests per `testing-strategy` (lowest pyramid level that proves the criterion; cover critical paths, error handling, edge cases, security boundaries; skip trivial code); run checks as CI does. With no CI, paste each check command with its exit code and output summary.
- A failing check or a bug found mid-issue gets `debug` (reproduce, isolate, root cause, regression test), not retries or symptom patches. Flaky tests unrelated to the change become follow-ups.
- Update docs the change makes stale (README, runbook, API docs): link, don't duplicate.
- Migrations: next free number, additive only.
- One commit per concern inside the PR (behaviour, tests, docs, generated files, formatting) so it reads commit by commit. Generated files are marked `linguist-generated=true` in `.gitattributes` so GitHub collapses them.
- PR body: closing keyword ("Fixes ABC-N"; "Part of" if partial), what changed, how each claim was verified (file:line or test), **Needs human**, follow-ups, check results.

## 4. Gate
- Once CI is green, run the `pr-gate` skill on the PR. It is the only review layer: it collects the evidence (checks, screenshots), runs the fresh-context and security reviews, posts look-here comments, sets the `risk:low|medium|high` label from the repo's `.claude/risk-paths.md` and its floor script, and posts the gate comment. Blocking findings go back to the implementer, who fixes or rebuts each on its thread; the gate's three-round cap applies.
- Gate a stacked PR on its own commits, and re-gate it if a rebase changes its diff.
- When the gate passes, `gh pr ready N`. A PR stays a draft until then.
- Unresolved design disagreements go to the user.
- If `pr-gate` isn't installed, follow its steps by hand: the tier floor comes from the sensitive paths, you may raise a tier but never lower it, and an agent's approval never counts as a person's.

## 5. Merge
- Only with green CI (or recorded checks), all threads resolved and a pr-gate label, in dependency order, by the repo's convention (default `gh pr merge --squash --delete-branch`). Rebase dependents after each merge.
- **Who merges is set by the tier:** you merge `risk:low` only. `risk:medium` and `risk:high` go to the user with the gate comment's "For you to look at". Never bypass branch rules (`--admin`). If the rules need an approval the agents can't give, hand low PRs over too, marked "no review needed".
- Before merging a PR that triggers a production deploy or carries a migration or feature flag, run `deploy-checklist` for it and put the rollback triggers in the PR. Watch triggered deploys; if one fails or trips a rollback trigger, stop merging that repo, report severity, impact and status in a few lines, and leave rollback to the user unless the profile allows it.
- Don't merge DO NOT MERGE or human-gated PRs, whatever their tier. A `needs-visual-review` PR (any visible UI or public content change) is at least medium, unless the gate finds that only wording changed. Hand it to the user with the gate's screenshots and how to preview locally.

## 6. Tracker
- Issues go to In Progress at start and Done on merge (unless automation does it), with a comment linking the PR.
- Follow-ups: one backlog issue each in the tracker only (no code TODOs), labelled per the rules, linked to the source, deduplicated. Tag debt follow-ups with a category (code, architecture, test, dependency, documentation, infrastructure) and score them (Impact + Risk) × (6 − Effort) to set priority where the tracker has one. If a create is refused, report it; don't retry. No tracker: list them in the report.

## 7. Wave digest
The digest is the only thing the user has to read. PRs grouped by tier, highest first; per PR one line: number, title, what to judge, links to the gate comment and its evidence. Low PRs the agent merged get a line each with nothing to judge. Then: merged and reverted counts per tier (pr-gate's step 7) · Actions minutes the wave used, summed from each run's `gh api repos/{owner}/{repo}/actions/runs/<id>/timing` when available · what's open and why · what needs the user (human-gated steps, then questions, then decisions) · follow-ups filed · whether the parent can close.
