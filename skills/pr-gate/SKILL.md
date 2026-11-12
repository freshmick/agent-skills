---
name: pr-gate
description: Verify a pull request and send it down one of two lanes before anyone merges it. Sends back a PR that is not one vertical slice, collects evidence (checks with exit codes), checks every changed screen itself from screenshots, runs one review pass with one or two fresh-context reviewers, set by the scope of the change, picks the lane from the repo's `lane:` setting and those paths, labels the PR and posts one gate comment. The agent merges agent-lane PRs itself; person's-lane PRs go to the user with 1-3 look-here points. Use whenever a PR is about to be handed over or merged, when the user says "/pr-gate", "gate this PR", "classify this PR", "is this PR safe to merge", or when /ship-wave reaches its gate or merge step.
argument-hint: "[PR numbers or URLs; default: the current branch's PR]"
---

# PR gate

The user can't read every diff. This gate decides whether a PR needs a person at all, and gives that human exactly what to look at. Input: `$ARGUMENTS` (one or more PRs). Run the steps per PR; independent PRs can run in parallel.

The core rules:
- **One PR, one slice.** A PR that isn't one vertical slice goes back before anyone reviews it.
- **The change sets the lane, not the author.** Look at what the diff touches, not at who or what wrote it.
- **The sensitive paths are the floor.** Your judgement may move a PR to the person's lane, never to the agent lane.
- **An agent's approval never counts as a person's.** That includes yours, a reviewer subagent's, and any GitHub approval from an account the agents share with the user.
- **Merge only agent-lane PRs yourself.** Permission rules can't read lanes, so this rule holds only if you hold to it.

## 0. Setup (once per repo)
- Read the repo's CLAUDE.md / AGENTS.md: the check commands, the merge convention (squash or rebase), branch rules, and where test servers may run.
- Lane setting and paths: the repo's `.claude/risk-paths.md` if it has one, else [references/risk-paths-template.md](references/risk-paths-template.md). A line that is exactly `lane: mvp` or `lane: live` is the repo setting; no such line, or no file, means `live`. Its `## sensitive` list (`## high` in older files) is the sensitive paths, and `## docs and tests` (`## low` in older files) the docs and tests. In older files a file matching a `## medium` glob is never docs or tests. Say in the gate comment which file and setting you used.
- Make sure the labels exist: `gh label create lane:agent --color 0e8a16`, `lane:person` `d93f0b` (with `--force` to make it safe to re-run).
- Read `main`'s protection (`gh api repos/{owner}/{repo}/rulesets` and branch protection). If a merge needs an approving review the agent cannot give, you cannot merge even agent-lane PRs: hand them over as "agent lane, no look needed" (step 6). On GitHub Free a private repo has no branch protection or rulesets, so nothing but this gate enforces the lanes; say so once in the gate comment.
- Identity: if `AGENT_GITHUB_LOGIN` is set (the agents' machine user), check that `gh api user -q .login` and the PR's commit authors (`gh pr view N --json commits -q '.commits[].authors[].login'`) match it. A mismatch goes in the gate comment and means you hand agent-lane PRs to the user instead of merging. If it is not set, the gate comment says the gate ran as the user's own account. Never stop for either case.

## 1. Scope the PR
- `gh pr view N --json title,body,files,additions,deletions,baseRefName,headRefName,labels,commits,reviewDecision,statusCheckRollup`, plus the issue it closes.
- **Stacked PRs:** if the body says "Builds on #M" or the base contains another PR's commits, gate only this PR's own commits (`git diff <parent-head>...<head>`). The parent gets its own gate. After the parent merges and this branch is rebased, check that the gated diff didn't change. If it did, gate again.
- List the changed files and classify each one: docs, tests, user-visible text, UI behaviour/layout, engine/logic, sensitive (matches a sensitive path; this wins over every other class), generated (goldens, lockfiles, snapshots).
- **Slice check.** Stop and send the PR back when it changes more than about 200 hand-written lines outside generated files or more than 8 files (the repo's instructions can set another cap; a batch of same-class docs or tests issues has the line cap only), when it closes more than one issue (a batch of same-class docs or tests issues is fine), or when its commits are layers that only work together (types, then templates, then engine, then commands). Label nothing. Comment once with the reason and a suggested cut into vertical slices, each working end to end on its own, and tell the user. Don't review it.
- **Paths.** Note whether the PR touches any sensitive path, whether every changed file is docs, tests or regenerated, and any needs-ADR match.
- **CI weakened.** Read the diff for a check step removed, a threshold lowered, an ignore or exclude list grown, a suppression or skip added (`# noqa`, `# type: ignore`, `eslint-disable`, `.skip`, `xfail`); an existing test deleted or its assertion changed, unless the issue asked for it. List each one with file:line.

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
One review pass. The scope of the change sets how many reviewers read it, whatever the lane.
- **Docs or tests only** (every file docs, tests or regenerated from an unchanged source, none sensitive): no reviewer. Read the diff yourself and confirm it changes no behaviour. If it does, it is not docs or tests only: treat it as ordinary in steps 3 and 5.
- **Ordinary (neither docs-or-tests only nor sensitive):** one fresh-context reviewer.
- **Sensitive, a stored or API data shape change, or a one-way door:** two fresh-context reviewers, the second with the security lens (step 4). No other review tools: never `/code-review` or `/security-review` from the gate; `/code-review ultra` is the user's to launch.

**Fresh-context review.** Spawn a reviewer subagent that didn't write the code and hasn't seen your reasoning. Give it read-only tools (Read, Grep, Glob, read-only git/gh). Give it the PR number, the diff range from step 1, the issue's acceptance criteria, the repo rules and the repo's `REVIEW.md` if there is one (what failure looks like in this repo, paths to skip, what counts as evidence), but not your opinion of the PR. Ask for blocking findings only:
- a bug in the changed lines, with the concrete input that produces the wrong result or the error;
- an acceptance criterion the PR doesn't meet, or the test that proves it missing;
- a security hole: injection, missing authorization, secrets, data loss.
Nothing else: no style, naming or structure remarks, no "consider handling", no requests for more tests, no findings about code the PR didn't change. Each finding has file:line and the failure scenario.

**Keep the PR the size it was.** A finding that isn't blocking by the list above is dropped: not a commit, not an issue, not a line in the gate comment.

**Verify before posting.** Each finding comes with its failure scenario; run it yourself in the PR worktree (the input, the request, the test) before it goes on the PR. A finding with nothing to run (a committed secret, a missing authorization check, a missing test) is confirmed by reading the lines it names. A finding you can't reproduce is dropped, not posted as "possible". Then post the survivors as review comments. Each must be fixed (with a test) or answered with a reason before the gate passes. Re-review only the changed lines. Never re-raise a finding already answered with a reason on its thread; if the reason is wrong, reply on that thread. After three rounds, stop and take the open point to the user.

## 4. Second reviewer
Sensitive PRs only (step 3). Spawn a second fresh-context reviewer with the same prompt and tools, told to look first for injection, missing authorization, secrets and data loss. Merge the two lists, drop duplicates, then verify and post as step 3 says. Record "second reviewer: no findings" or the finding and how it was resolved.

## 5. Pick the lane
The repo setting (step 0) and the paths (step 1) set the lane:

| Setting | Agent lane: the agent merges | Person's lane: the user merges |
|---|---|---|
| `mvp` | every other PR | PRs touching a sensitive path, or marked `Door: one-way` (in the body, or you find it is one) |
| `live` | docs or tests only PRs (step 3) | every other PR |

A PR goes to the person's lane whatever the setting when it is a one-way door, when step 1 found CI weakened and the PR body doesn't explain it (name each finding under "Look at"), or when a needs-ADR path changed and the PR carries no ADR (a `docs/adr/*.md` file in the diff or linked from the body; ask for it). Your judgement may also move a PR to the person's lane, for example a stored or API data shape change, or a risky path the tests miss; never the other way. When in doubt, the person's lane: a wrong call there costs the user a minute, and a wrong agent merge ships a bug unseen.

## 6. Label, comment, merge
- Set exactly one lane label: `lane:agent` or `lane:person`, removing the other and any old `risk:*` label.
- Post **one** gate comment using [references/gate-comment.md](references/gate-comment.md). On a re-run, edit your earlier gate comment instead of adding a new one (find it by its first line `<!-- pr-gate -->`).
- **No re-gate** after a rebase onto the base that leaves the PR's own diff unchanged; the earlier gate stands once CI is green again. After the rebase, edit the gate comment's head sha and re-post the look-here comments on the new head. CI must run on the rebased head before the merge. A red run there goes back to the implementer with the PRs merged since the branch point and the files they touched. The review stands; only the fix's changed lines are re-reviewed.
- **Agent lane:** if checks, screens and the reviews step 3 called for are clean and no thread is open, rebase onto the default branch, wait for CI on that head, then merge by the repo's convention (`gh pr merge N --squash --delete-branch` or `--rebase`; never `--admin`). Then rebase each stacked dependant: `git rebase --onto origin/<default> origin/<parent-branch> <branch>`, `git push --force-with-lease`, and re-run its gate if its diff changed. If `main`'s rules need an approval the agent can't give, don't bypass them. Tell the user "agent lane, ready, no review needed: just merge".
- **Person's lane:** don't merge. Post one review comment at each of the one to three decision points: `gh api repos/{owner}/{repo}/pulls/N/comments -f commit_id=<head sha> -f path=<file> -F line=<n> -f body=...`, body starting `<!-- pr-gate look-here -->` and saying in one or two sentences what to judge there. Then put a `## Look here` section at the top of the PR body listing those points with permalinks (`https://github.com/{owner}/{repo}/blob/<head sha>/<path>#L<a>-L<b>`). On a re-run replace that section and delete your stale look-here comments. More than three points means the PR is too big: say so and suggest the cut instead of listing them all. Tell the user in plain words that the PR is ready, why it is in their lane, and the 1-3 things to look at, with links (the review summary, and for sensitive PRs the decision only a person can make). Look-here points are decisions, never "check this screen".
- A GitHub approval from the agents' shared account proves nothing. If the repo relies on it, suggest a bot account or GitHub App for the agents, plus branch protection that requires the CI checks.

## 7. Report
Per PR, one line: number, lane, why, state (merged / waiting for the user / sent back to slice / blocked by X). Then, under "Needs you": what the user must look at or merge, most urgent first.

