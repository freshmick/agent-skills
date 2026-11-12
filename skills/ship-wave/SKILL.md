---
name: "ship-wave"
description: Lead a wave of issues to merged PRs in any repo(s) with any tracker (Linear, GitHub Issues, Jira, or a plain list). Parallel sub-agents implement as draft PRs, pr-gate reviews each one and picks its lane, PRs merge in dependency order, and the tracker stays current. Use when the user says "/ship-wave", passes a handoff prompt, or asks to implement/kick off a set of issues.
argument-hint: "[issue IDs, project/milestone, or path to a handoff prompt]"
---

# Ship a wave

You are the lead engineer. Issues are the spec; this skill is the rules. Input: `$ARGUMENTS` (issue IDs, a project/milestone, or a handoff prompt path). A handoff prompt overrides this skill where they conflict.

**Engineering skills.** Steps below name skills from the engineering plugin (`debug`, `deploy-checklist`, `architecture`). Review skills are not loaded here: pr-gate owns review. Load each with the Skill tool when its step starts, and tell sub-agents which one to load. They supply the lens; this skill and the repo's rules still decide scope, format and where output goes. If one isn't installed, the short checklist given in that step is the fallback.

## 0. Setup
Settle these from, in priority order: the handoff prompt, a matching `profiles/*.md` next to this file, `.claude/ship-wave.md`, the repos' AGENTS.md/CLAUDE.md/CI config. Ask the user only for what's left.
- Tracker (Linear, GitHub Issues, Jira, none) · repos and code host · worktree root (default `<repo>/../worktrees/`) · checks CI runs · merge convention and what merges trigger · repo-specific rules.
- Identity: if `AGENT_GITHUB_LOGIN` is set (the agents' machine user), confirm `gh api user -q .login` matches it and that commits will carry it. If it is not set, say plainly in the plan message that this wave runs as the user's own GitHub account, then continue.

With no profile and repeat waves likely, offer to save one.

## 1. Before starting
- Read the instruction files of every touched repo, and every issue in full, including comments and blocking links. Don't re-decide what they settle. Skip an issue someone else already has In Progress (on GitHub Issues: assigned to someone else).
- `git fetch` all repos. Never edit main checkouts.
- Put ambiguities (especially product behaviour) in one numbered md file with a/b/c options and a recommendation; give the user its path. For technical design choices, write each option the way `architecture` does (context, options with pros/cons, trade-off, consequences), kept to a few lines each. Don't block issues that don't depend on the answers.

## 2. Plan
- In a repo with no code yet, the first issue is a walking skeleton: the smallest code base that builds, runs one path end to end and passes CI. Everything else waits for it. Order the others by blocking links and code dependencies; parallelize the rest, but serialize issues whose listed files overlap (and the usual shared files: styles, layouts, config, lockfiles).
- Batch same-class docs or tests issues into one PR per wave; the batch has the line cap but not the file cap. Everything else stays one issue, one PR. Split an issue only when it mixes a contract or sensitive change with ordinary code: that part gets its own PR. Before opening any PR, `gh pr list --search "<issue id>"` for one that already exists.
- An issue over the size cap (about 200 hand-written lines outside generated files such as goldens, lockfiles and snapshots, across at most 8 files, unless the repo sets another), or one that works only once several layers change together, is not a slice. Don't slice it here: hold that issue, tell the user why and point them to `to-tickets`. Start the others.
- **At most 3 PRs wait on the user at once** (gated into the person's lane, not yet merged). Don't start an issue that would push past that; start it when one of them lands. Agent-lane PRs don't count.
- Work out each issue's likely lane the way pr-gate's steps 1 and 5 do, over the files it will likely touch: the repo's `.claude/risk-paths.md` (its `lane:` line; none means `live`), else pr-gate's `references/risk-paths-template.md`. If an issue touches a sensitive path or needs an ADR, write a one-page proposal for it (plan-mode output is the format: what changes, which contract, architecture or sensitive paths, the options and a recommendation) and wait for the user's approval of that issue. Start the others right away.
- Tell the user the plan in a few lines, including which issues wait for a proposal.

## 3. Implement (one sub-agent per issue)
- One issue, one fresh worktree from the updated default branch, one small PR opened as a draft (`gh pr create --draft`). Branch: the tracker's suggested name, else `<issue-id>-<short-slug>`.
- Only the acceptance criteria. Anything else you notice is left out of the PR and gets one line in the wave digest (step 7), nothing more: no commit, no issue, no TODO.
- Follow the repo's conventions and instructions (pass the step 0 rules to each sub-agent). Touch only the files the criterion needs (the issue lists them; a file it doesn't list gets a sentence in the PR body). No new abstraction, interface or helper unless this slice uses it twice. No cleanup of code the slice doesn't change. Small single-purpose functions, clear names, config from env, no secrets committed.
- Tests: the one test the issue names, at the lowest pyramid level that proves the criterion, plus one per error case the issue lists, plus a regression test for each bug fixed on the way. No more. When the slice changes code that has no tests, the PR's first commit is a test pinning today's behaviour; no separate issue. Run checks as CI does. With no CI, paste each check command with its exit code and output summary.
- **Review before opening the PR:** run pr-gate's fresh-context review (its step 3 reviewer prompt, same read-only tools, blocking findings only) on your own diff and fix what it finds. The gate runs it again; arriving pre-reviewed is what keeps the gate to one round.
- **Measure before opening the PR:** `git diff --stat origin/<default>...HEAD` (for a stacked branch, against the parent branch), minus generated files. Over the cap (step 2), don't open the PR: stop, report the size and where a cut would fall, and let the lead hold the issue for `to-tickets`.
- A failing check or a bug found mid-issue gets `debug` (reproduce, isolate, root cause, regression test), not retries or symptom patches. Flaky tests unrelated to the change get a line in the digest, not a fix in this PR.
- Update docs the change makes stale (README, runbook, API docs): link, don't duplicate.
- Migrations: next free number, additive only.
- One commit per concern inside the PR (behaviour, tests, docs, generated files, formatting) so it reads commit by commit. Generated files are marked `linguist-generated=true` in `.gitattributes` so GitHub collapses them.
- PR body: short enough to read in a minute. Closing keyword ("Fixes ABC-N"; "Part of" if partial), then a few bullets on what changed and why, the one command or test that proves the criterion, and one line `Door: two-way` (a revert undoes it) or `Door: one-way` (a migration, a deleted field, an external call) with the reason. No "Needs human" section (review is what a PR is for) and no check logs (the gate comment carries the evidence).

## 4. Gate
- Once CI is green, run the `pr-gate` skill on the PR. It is the only review layer: it collects the evidence (checks, screenshots), runs the reviews the sensitive paths call for, picks the lane, posts look-here comments and the gate comment. A PR it sends back as not one slice goes back to `to-tickets`, not to more commits. Blocking findings go back to the implementer, who fixes or rebuts each on its thread; the gate's three-round cap applies.
- Gate a stacked PR on its own commits, and re-gate it if a rebase changes its diff.
- When the gate passes, `gh pr ready N`. A PR stays a draft until then.
- Unresolved design disagreements go to the user.
- If `pr-gate` isn't installed, follow its steps by hand: the lane comes from the `lane:` setting and the sensitive paths, you may move a PR to the person's lane but never the other way, and an agent's approval never counts as a person's.

## 5. Merge
- Only with green CI (or recorded checks), all threads resolved and a pr-gate lane label, in dependency order, by the repo's convention (default `gh pr merge --squash --delete-branch`). Rebase dependents after each merge.
- **Who merges is set by the lane:** you merge `lane:agent` only. `lane:person` PRs go to the user with the gate comment's "Look at". Never bypass branch rules (`--admin`). If the rules need an approval the agents can't give, hand agent-lane PRs over too, marked "no review needed".
- Before merging a PR that triggers a production deploy or carries a migration or feature flag, run `deploy-checklist` for it and put the rollback triggers in the PR. Watch triggered deploys; if one fails or trips a rollback trigger, stop merging that repo, report severity, impact and status in a few lines, and leave rollback to the user unless the profile allows it.
- Don't merge DO NOT MERGE or human-gated PRs, whatever their lane. pr-gate checks the screens of a `needs-visual-review` PR (any visible UI or public content change) itself; hand it over with the gate's findings, never with screenshots for the user to judge.

## 6. Tracker
- Issues go to In Progress at start and Done on merge (unless automation does it), with a comment linking the PR. The tracker is the shared record; the lead also appends one line per issue (issue, PR, gate result, merged or not) to `<worktree root>/ship-wave-ledger-<date>.md`, never committed, and re-reads it after a context compaction.
- Agents create no issues from reviews or implementation. Anything noticed outside a PR's criteria goes under "Left out" in the digest, one line each; the user decides what becomes an issue.

## 7. Wave digest
The digest is the only thing the user has to read. PRs grouped by lane, person's lane first; per PR one line: number, title, what to judge, links to the gate comment and its evidence. Agent-lane PRs the agent merged get a line each with nothing to judge. Then: merged and reverted counts per lane (pr-gate's step 7) · Actions minutes the wave used, summed from each run's `gh api repos/{owner}/{repo}/actions/runs/<id>/timing` when available · what's open and why · what needs the user (human-gated steps, then questions, then decisions) · left out: one line each, for the user to turn into issues or drop · recurring findings: a finding the gate raised on two or more PRs this wave, with where it belongs instead (a lint rule, a hook, CLAUDE.md or REVIEW.md, or a skill), never as a comment on the next PR · whether the parent can close.
