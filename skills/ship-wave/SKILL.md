---
name: "ship-wave"
description: Lead a wave of issues to merged PRs in any repo(s) with any tracker (Linear, GitHub Issues, Jira, or a plain list). Parallel sub-agents implement as draft PRs, pr-gate reviews, lanes and merges each one, PRs land in dependency order, and the tracker stays current. Use when the user says "/ship-wave", passes a handoff prompt, or asks to implement/kick off a set of issues.
license: MIT
argument-hint: "[issue IDs, project/milestone, or path to a handoff prompt]"
---

# Ship a wave

You are the lead engineer. Issues are the spec; this skill is the rules. Input: `$ARGUMENTS` (issue IDs, a project/milestone, or a handoff prompt path). A handoff prompt overrides this skill where they conflict.

`pr-gate` must be installed next to this skill: it owns review, lanes, rebase and merge. `pr` from Matt Pocock's plugin writes the PR body. Two skills from the engineering plugin are optional: `debug` for a failing check or a bug found mid-issue, and `deploy-checklist` before a merge that deploys or migrates. Load one with the Skill tool when its step starts, and tell sub-agents which to load; the short checklist in that step is the fallback when it isn't installed.

## 0. Setup
Settle these from, in priority order: the handoff prompt, `.claude/ship-wave.md`, the repos' CLAUDE.md/AGENTS.md (with the `docs/agents/issue-tracker.md` it points at, written by `/setup-matt-pocock-skills`) and CI config. Ask the user only for what's left.
- Tracker (Linear, GitHub Issues, Jira, none) · repos and code host · worktree root (default `<repo>/../worktrees/`) · checks CI runs · merge convention and what merges trigger · repo-specific rules, including the slice cap when the repo sets one.
- How many PRs may wait on the user at once. It changes from wave to wave, so take it only from the handoff prompt; otherwise ask every wave, offering 3 (the default, for when the user is around) or no limit (for a wave left running overnight).
- Identity, as pr-gate step 0: if `AGENT_GITHUB_LOGIN` is set (the agents' machine user), confirm `gh api user -q .login` matches it and that commits will carry it. If it is not set, say plainly in the plan message that this wave runs as the user's own GitHub account, then continue.

With no `.claude/ship-wave.md` and repeat waves likely, offer to write one. With no `.claude/risk-paths.md`, say so once and suggest `/risk-paths`; the wave runs on pr-gate's template with `lane: live` until the file exists.

## 1. Before starting
- Read the instruction files of every touched repo, and every issue in full, including comments, the parent spec and blocking links. Don't re-decide what they settle. Skip an issue someone else already has In Progress (on GitHub Issues: assigned to someone else).
- `git fetch` all repos. Never edit main checkouts.
- Put ambiguities (especially product behaviour) in one numbered md file with a/b/c options and a recommendation; give the user its path. A technical design choice gets its context, the options with pros and cons, and the recommendation, a few lines each. Don't block issues that don't depend on the answers.

## 2. Plan
- In a repo with no code yet, the first issue is a walking skeleton: the smallest code base that builds, runs one path end to end and passes CI. Everything else waits for it. Order the others by blocking links and code dependencies; parallelize the rest, but serialize issues whose listed files overlap (and the usual shared files: styles, layouts, config, lockfiles).
- Batch same-class docs or tests issues into one PR per wave. Everything else stays one issue, one PR. Split an issue only when it mixes a contract or sensitive change with ordinary code: that part gets its own PR. Before opening any PR, `gh pr list --search "<issue id>"` for one that already exists.
- An issue over pr-gate's slice cap (its step 1), or one that works only once several layers change together, is not a slice. Don't slice it here: hold that issue, tell the user why and point them to `to-tickets`. Start the others.
- **Respect the limit from setup on PRs waiting on the user** (gated into the person's lane, not yet merged). Don't start an issue that would push past it; start it when one of them lands. Agent-lane PRs don't count.
- Work out each issue's likely lane the way pr-gate's steps 1 and 5 do, over the files it will likely touch. If an issue touches a sensitive path or needs an ADR, write a one-page proposal for it (what changes, which contract, architecture or sensitive paths, the options and a recommendation) and wait for the user's approval of that issue. Start the others right away.
- Tell the user the plan in a few lines, including which issues wait for a proposal.

## 3. Implement (one sub-agent per issue)
- One issue, one fresh worktree from the updated default branch, one small PR opened as a draft (`gh pr create --draft`). Branch: the tracker's suggested name, else `<issue-id>-<short-slug>`.
- Only the acceptance criteria. Anything else you notice is left out of the PR and gets one line in the wave digest (step 6), nothing more: no commit, no issue, no TODO.
- Follow the repo's conventions and instructions (pass the step 0 rules to each sub-agent). Touch only the files the criterion needs (the issue lists them; a file it doesn't list gets a sentence in the PR body). No cleanup of code the slice doesn't change.
- Tests: the one test the issue names, at the lowest pyramid level that proves the criterion, plus one per error case the issue lists, plus a regression test for each bug fixed on the way. No more. When the slice changes code that has no tests, the PR's first commit is a test pinning today's behaviour; no separate issue. Write the slice's acceptance test first and run it to see it fail before writing code. Never edit, delete or skip an existing test; when a test and the issue disagree, stop and report the conflict to the lead (on the PR once there is one) instead of making the test pass. The whole suite runs before the PR opens, as CI runs it; the gate records the evidence.
- **Measure before opening the PR:** `git diff --stat origin/<default>...HEAD` (for a stacked branch, against the parent branch), minus generated files. Over pr-gate's cap, don't open the PR: stop, report the size and where a cut would fall, and let the lead hold the issue for `to-tickets`.
- A failing check or a bug found mid-issue gets `debug` (reproduce, isolate, root cause, regression test), not retries or symptom patches. Flaky tests unrelated to the change get a line in the digest, not a fix in this PR.
- Update docs the change makes stale (README, runbook, API docs): link, don't duplicate.
- One commit per concern inside the PR (tests, behaviour, docs, generated files, formatting) so it reads commit by commit.
- PR body: the `pr` skill's template, short enough to read in a minute. The closing keyword comes first ("Fixes ABC-N"; "Part of" if partial). Evidence is the one command or test that proves the criterion; the gate comment carries the check logs and screenshots. The Door line is `two-way` (a revert undoes it) or `one-way` (a migration, a deleted field, an external call), with the reason. No "Needs human" section: review is what a PR is for.

## 4. Gate and merge
- Run the `pr-gate` skill on each PR as soon as its implementer reports it. It is the only review layer and the only merger: it collects the evidence, runs the reviews the paths call for, picks the lane, posts the look-here comments and the gate comment, merges agent-lane PRs, and rebases the dependants after a merge. You never merge, rebase or re-gate by hand.
- Gate in dependency order, so merges land in it. A stacked PR is gated on its own commits; pr-gate re-gates it if a rebase changes its diff.
- A PR the gate sends back as not one slice goes back to `to-tickets`, not to more commits. Blocking findings, and a red run after a rebase, go back to the implementer with what the gate reports; the implementer fixes or rebuts each finding on its thread, and the gate's three-round cap applies.
- When the gate passes, `gh pr ready N`. A PR stays a draft until then.
- Unresolved design disagreements go to the user.
- `lane:person` PRs go to the user with the gate comment's "Look at". Don't merge DO NOT MERGE or human-gated PRs, whatever their lane. If the repo's rules need an approval the agents can't give, pr-gate hands agent-lane PRs over too, marked "no review needed".
- Before pr-gate merges a PR that triggers a production deploy or carries a migration or feature flag, run `deploy-checklist` for it and put the rollback triggers in the PR. Watch triggered deploys; if one fails or trips a rollback trigger, stop the wave in that repo, report severity, impact and status in a few lines, and leave rollback to the user unless `.claude/ship-wave.md` allows it.
- pr-gate checks the screens of a visible UI or public content change itself; hand such a PR over with the gate's findings, never with screenshots for the user to judge.

## 5. Tracker
- Issues go to In Progress at start and Done on merge (unless automation does it), with a comment linking the PR. The tracker is the shared record; the lead also appends one line per issue (issue, PR, gate result, merged or not) to `<worktree root>/ship-wave-ledger-<date>.md`, never committed, and re-reads it after a context compaction.
- Agents create no issues from reviews or implementation. Anything noticed outside a PR's criteria goes under "Left out" in the digest, one line each; the user decides what becomes an issue.

## 6. Wave digest
The digest is the only thing the user has to read. PRs grouped by lane, person's lane first; per PR one line: number, title, what to judge, links to the gate comment and its evidence. Agent-lane PRs the gate merged get a line each with nothing to judge. Then: what's open and why · what needs the user (human-gated steps, then questions, then decisions) · left out: one line each, including reviewer findings outside an issue's criteria, for the user to turn into issues or drop · recurring findings: a finding the gate raised on two or more PRs this wave, proposed as a check first, in this order: a Semgrep rule (in the repo's Semgrep rules folder when it has one), a lint rule, a hook or CI step; a CLAUDE.md or REVIEW.md line, or a fix to the skill text that caused it, only when no check can express it, with the reason; never a comment on the next PR · whether the parent can close.
