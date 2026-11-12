---
name: pr-gate
description: Verify a pull request and give it a risk tier (low, medium, high) before anyone merges it. Sends back a PR that is not one vertical slice, collects evidence (checks with exit codes), checks every changed screen itself from screenshots, runs a fresh-context review and a security review, sets the tier from the repo's sensitive paths, labels the PR risk:low/medium/high and posts one gate comment. The agent merges low-risk PRs itself; medium and high go to the user with the evidence. Use whenever a PR is about to be handed over or merged, when the user says "/pr-gate", "gate this PR", "classify this PR", "is this PR safe to merge", or when /ship-wave reaches its gate or merge step.
argument-hint: "[PR numbers or URLs; default: the current branch's PR]"
---

# PR gate

The user can't read every diff. This gate decides how much human attention each PR needs, and gives that human exactly what to look at. Input: `$ARGUMENTS` (one or more PRs). Run the steps per PR; independent PRs can run in parallel.

The core rules:
- **One PR, one slice.** A PR that isn't one vertical slice goes back before anyone reviews it.
- **The change sets the tier, not the author.** Look at what the diff touches, not at who or what wrote it.
- **The sensitive paths are the floor.** Your judgement may raise a tier and may never lower it.
- **An agent's approval never counts as a person's.** That includes yours, a reviewer subagent's, and any GitHub approval from an account the agents share with the user.
- **Merge only `risk:low` yourself.** Permission rules can't read labels, so this rule holds only if you hold to it.

## 0. Setup (once per repo)
- Read the repo's CLAUDE.md / AGENTS.md: the check commands, the merge convention (squash or rebase), branch rules, and where test servers may run.
- Sensitive paths: the repo's `.claude/risk-paths.md` if it has one, else [references/risk-paths-template.md](references/risk-paths-template.md). Say in the gate comment which one you used.
- Make sure the labels exist: `gh label create risk:low --color 0e8a16`, `risk:medium` `fbca04`, `risk:high` `d93f0b`, `reverted` `b60205` (with `--force` to make it safe to re-run).
- Read `main`'s protection (`gh api repos/{owner}/{repo}/rulesets` and branch protection). If a merge needs an approving review the agent cannot give, you cannot merge even low PRs: hand them over as "low, no look needed" (step 6). On GitHub Free a private repo has no branch protection or rulesets, so nothing but this gate enforces the tiers; say so once in the gate comment.
- Identity: if `AGENT_GITHUB_LOGIN` is set (the agents' machine user), check that `gh api user -q .login` and the PR's commit authors (`gh pr view N --json commits -q '.commits[].authors[].login'`) match it. A mismatch goes in the gate comment and means you hand low PRs to the user instead of merging. If it is not set, the gate comment says the gate ran as the user's own account. Never stop for either case.

## 1. Scope the PR
- `gh pr view N --json title,body,files,additions,deletions,baseRefName,headRefName,labels,commits,reviewDecision,statusCheckRollup`, plus the issue it closes.
- **Stacked PRs:** if the body says "Builds on #M" or the base contains another PR's commits, gate only this PR's own commits (`git diff <parent-head>...<head>`). The parent gets its own gate. After the parent merges and this branch is rebased, check that the gated diff didn't change. If it did, gate again.
- List the changed files and classify each one: docs, tests, user-visible text, UI behaviour/layout, engine/logic, sensitive (matches the sensitive paths), generated (goldens, lockfiles, snapshots).
- **Slice check.** Stop and send the PR back when it changes more than about 200 hand-written lines outside generated files or more than 8 files (the repo's instructions can set another cap; a batch of same-class low-tier issues has the line cap only), when it closes more than one issue (a batch of same-class low-tier issues is fine), or when its commits are layers that only work together (types, then templates, then engine, then commands). Label nothing. Comment once with the reason and a suggested cut into vertical slices, each working end to end on its own, and tell the user. Don't review it.
- **Floor.** Each changed file takes the highest tier among the globs it matches, and a file matching none is medium. The floor is the highest tier of any file; low only when every file is low. Note any needs-ADR match.
- **CI weakened.** Read the diff for a check step removed, a threshold lowered, an ignore or exclude list grown, a suppression or skip added (`# noqa`, `# type: ignore`, `eslint-disable`, `.skip`, `xfail`). List each one with file:line.

## 2. Collect evidence
Work in a worktree at the PR head, never the main checkout.
- **GitHub checks:** `gh pr checks N`. Every required check must be green, and any pending check blocks.
- **Local checks:** run the repo's check commands (lint, format, tests, type check, API/schema drift). Record each command, its exit code and a one-line result ("961 passed"). Re-run a failure on its own before calling it real (parallel suites time out).
- **Screens:** for any change a user can see, start the app on a free port (respect the repo's forbidden ports). Use Playwright to screenshot each changed screen: before (base) and after (head) when the change is visual, and after only for new screens. Use phone width too if the change mentions phones or layout. Publish the images so the user can open them from the PR, in this order of preference:
  - a private evidence page (the Artifact tool, if available): one page per PR or per batch, with before/after side by side and a line on what to judge in each; link it from the gate comment;
  - else the evidence branch `pr-evidence` (an orphan branch that never merges; folder `pr-<N>/`), linked as `https://github.com/{owner}/{repo}/blob/pr-evidence/pr-<N>/<file>.png`;
  - else local paths.
  When gating a stack, screenshots from one combined branch with all the PRs applied are fine; say so in the comment.
- **Check the screens yourself.** Open every screenshot (the Read tool shows images) and judge it against the issue's acceptance criteria and the design: the change is there and does what the issue says, nothing else moved, no overlap, cut-off or missing text, no broken images, empty and error states where the change has them, and phone width when it applies. Each defect is a blocking finding with its screenshot. Record what you checked and what you found. Never hand a screenshot to the user to judge: the user decides product questions, not whether a screen looks right.
- Evidence is what you ran and saw. Never write "tests pass" without the command and its output.

## 3. Review
Depth follows the floor from step 1. If step 5 raises the tier, run what the new tier needs before step 6.
- **low:** no reviewer. Read the diff yourself and confirm it does only what a low file may do (docs, tests, wording, regenerated files). Anything else raises the tier to medium.
- **medium:** the fresh-context review below.
- **high:** the fresh-context review, then `/security-review` (step 4) and `/code-review high <PR>` via the code-review skill. Never launch `/code-review ultra` yourself: it is costly and user-triggered.

**Fresh-context review.** Spawn a reviewer subagent that didn't write the code and hasn't seen your reasoning. Give it read-only tools (Read, Grep, Glob, read-only git/gh). Give it the PR number, the diff range from step 1, the issue's acceptance criteria, the repo rules and the repo's `REVIEW.md` if there is one (what failure looks like in this repo, paths to skip, what counts as evidence), but not your opinion of the PR. Ask for blocking findings only:
- a bug in the changed lines, with the concrete input that produces the wrong result or the error;
- an acceptance criterion the PR doesn't meet, or the test that proves it missing;
- a security hole: injection, missing authorization, secrets, data loss.
Nothing else: no style, naming or structure remarks, no "consider handling", no requests for more tests, no findings about code the PR didn't change. Each finding has file:line and the failure scenario.

**Keep the PR the size it was.** A finding that isn't blocking by the list above is dropped: not a commit, not an issue, not a line in the gate comment.

**Verify before posting.** Each finding comes with its failure scenario; run it yourself in the PR worktree (the input, the request, the test) before it goes on the PR. A finding with nothing to run (a committed secret, a missing authorization check, a missing test) is confirmed by reading the lines it names. A finding you can't reproduce is dropped, not posted as "possible". Then post the survivors as review comments. Each must be fixed (with a test) or answered with a reason before the gate passes. Re-review only the changed lines. Never re-raise a finding already answered with a reason on its thread; if the reason is wrong, reply on that thread. After three rounds, stop and take the open point to the user.

## 4. Security review
High tier only (step 3). Run `/security-review` (the built-in skill) in the PR worktree against the PR's base. Any open finding blocks. Record "security review: no findings" or the finding and how it was resolved.

## 5. Set the tier
Start at the floor (step 1), then raise it if judgement says so:

| Tier | What it covers | Who merges |
|---|---|---|
| **low** | Docs, comments, tests only, user-visible wording that doesn't change layout or behaviour, generated files regenerated from an unchanged source | the agent, once steps 2-4 are clean |
| **medium** | Ordinary screen and feature changes, engine logic outside sensitive paths | the user, from the gate comment: what changed, what you checked (screens included) and any product decision; not the code |
| **high** | Anything on the sensitive paths: sign-in/auth, storage, data or tenant isolation, CI workflows, container/deploy config, dependencies and lockfiles, calculation formulas and their defaults, and rule files such as CLAUDE.md and the sensitive-path list itself | the user, after the full review (step 3: fresh-context, security, `/code-review high`) |

Raise the tier when:
- a "low" PR changes behaviour as well as words;
- a change alters a stored or API data shape (a field becomes optional, or a schema changes);
- the change is hard to revert (the PR body says `Door: one-way`, or you find it is one);
- the tests don't cover the risky path;
- you can't say what the checks would miss;
- step 1 found CI weakened: treat each finding as high unless the PR body explains it, and name it under "For you to look at";
- a needs-ADR path changed and the PR carries no ADR (a `docs/adr/*.md` file in the diff or linked from the body): raise to high and ask for the ADR.

When in doubt, go up one tier: a wrong "medium" costs the user a minute, and a wrong "low" ships a bug unseen.

## 6. Label, comment, merge
- Set exactly one label: remove other `risk:*` labels and add the new one.
- Post **one** gate comment using [references/gate-comment.md](references/gate-comment.md). On a re-run, edit your earlier gate comment instead of adding a new one (find it by its first line `<!-- pr-gate -->`).
- **No re-gate** after a clean merge of the base into the PR that leaves the PR's own diff unchanged; the earlier gate stands once CI is green again.
- **low:** if checks, review and security review are clean and no thread is open, merge by the repo's convention (`gh pr merge N --squash --delete-branch` or `--rebase`; never `--admin`). Then rebase each stacked dependant: `git rebase --onto origin/<default> origin/<parent-branch> <branch>`, `git push --force-with-lease`, and re-run its gate if its diff changed. If `main`'s rules need an approval the agent can't give, don't bypass them. Tell the user "low risk, ready, no review needed: just merge".
- **medium / high:** don't merge. Post one review comment at each of the one to three decision points: `gh api repos/{owner}/{repo}/pulls/N/comments -f commit_id=<head sha> -f path=<file> -F line=<n> -f body=...`, body starting `<!-- pr-gate look-here -->` and saying in one or two sentences what to judge there. Then put a `## Look here` section at the top of the PR body listing those points with permalinks (`https://github.com/{owner}/{repo}/blob/<head sha>/<path>#L<a>-L<b>`). On a re-run replace that section and delete your stale look-here comments. More than three points means the PR is too big: say so and suggest the cut instead of listing them all. Tell the user in plain words that the PR is ready, which tier, and the 1-3 things to look at, with links (the review summary, and for high the decision only a person can make). Look-here points are decisions, never "check this screen".
- **Reverted:** when a merged PR is undone (a revert PR merges, or its squash commit is reverted on `main`), add the `reverted` label to the original PR so the counts in step 7 stay honest.
- A GitHub approval from the agents' shared account proves nothing. If the repo relies on it, suggest a bot account or GitHub App for the agents, plus branch protection that requires the CI checks.

## 7. Report
Per PR, one line: number, tier, why, state (merged / waiting for the user / sent back to slice / blocked by X). Then, under "Needs you": what the user must look at or merge, most urgent first.

Then one line per tier for the repo: merged and reverted counts, from `gh pr list --state merged --label risk:<tier> --limit 500 --json number -q length` and the same with `--label reverted` added. This is the data that decides whether medium can merge without a person later.
