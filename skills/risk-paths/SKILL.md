---
name: risk-paths
description: Write a repo's `.claude/risk-paths.md`, the file pr-gate reads to decide which PRs an agent may merge and which go to a person. Sets the lane (`mvp` or `live`), finds candidate sensitive paths in the repo, confirms them with the user and opens the file as its own PR. Run once per repo, before the first wave, or when the lane or the sensitive paths change. Use when the user says "/risk-paths", "set up risk paths", "which paths are sensitive", or when ship-wave or pr-gate report that the repo has no `risk-paths.md`.
license: MIT
argument-hint: "[repo path; default: the current repo]"
---

# Risk paths

One file tells the gate where a mistake hurts. This skill writes it with the user. It does not set up checks, CI or a baseline; those are the repo's own business.

Input: `$ARGUMENTS` (a repo path, default the current one). Read [pr-gate's template](../pr-gate/references/risk-paths-template.md) first: it is the file format, the generic globs, and the rules the gate applies to each section.

## 1. The lane
Ask one question: is anyone real using this product? No real users means `mvp`: the agent merges every PR except those touching a sensitive path or marked a one-way door. Real users, or someone else's repo, means `live`: the agent merges docs, tests and regenerated files only. If the user is unsure, it is `live`.

## 2. Find the candidates
Look before asking. List the paths that match each of these, from the repo's tree and its instruction files, and keep the globs short:
- sign-in, sessions, tokens, permissions
- where user data is written, read and deleted
- anything that charges money or calls an external service
- CI and deploy: workflows, `CODEOWNERS`, Dockerfiles, compose files, infra
- dependency manifests and lockfiles
- stored data shape: migrations, schema files, API contracts, protobufs
- the checks themselves: lint, test and coverage config, pre-commit, Semgrep and secret-scanning config
- the rules agents follow: `CLAUDE.md`, `AGENTS.md`, `.claude/**`

Then show the list in two parts: the generic globs from the template that match something here, and the repo-specific candidates you found. Ask what is missing, one question: "Where else would a wrong change hurt a user, lose data or cost money?" Ask about `needs-adr` only if the repo has a `docs/adr/` folder or the user mentions decision records. Take the docs-and-tests globs from the template and add the repo's own test folders.

## 3. Write and open
- Write `.claude/risk-paths.md` from the template: the `lane:` line, then `## sensitive`, `## needs-adr` when it has entries, and `## docs and tests`. Drop the template's example globs that match nothing here. Keep the ` - ` notes short; they are for the next person who reads the file.
- Open it as its own PR on a branch named `risk-paths`, never a direct commit to the default branch. The file matches `.claude/**`, so pr-gate puts it in the person's lane: say so, and that the user merges it.
- Tell the user in a few lines what the gate will do now: which PRs merge without them, which come to them, and that changing the lane later is one line in this file.

When the file already exists, read it, show the diff you propose and change only what the user confirms.
