---
name: slice-work
description: Break an agreed design into vertical slices, one issue per slice, ready for ship-wave. Asks first when requirements or acceptance criteria are unclear. Use when the user says "break out the work", "break this down", "create the issues" or "slice this", or when a design discussion ends and implementation is next.
---

# Slice work

Input: the design agreed in this conversation, a design doc, or existing tracker items (epic, story, task). Output: issues in the tracker, one vertical slice each, in order, ready for /ship-wave.

## 1. Clarify first
Before cutting anything, check that each piece of the input says, so a test could prove it:
- who it is for and what they can do afterwards that they can't today;
- the acceptance criteria: observable outcomes, not implementation steps;
- the edge and error cases (empty, invalid, missing, too many, no permission);
- what is out of scope;
- limits that change the design: data shape, compatibility, performance, security, migration of existing data;
- dependencies on other work, repos or people.

Anything missing, vague ("fast", "handle errors", "support X") or contradictory is a question, not a guess. Ask them all at once: numbered, each with a/b/c options and a recommendation. Read the existing tracker items, their comments and the code first, so you only ask what they don't settle. Slice only what is clear; say which parts wait on which answers.

## 2. Find the slices
- Each slice is a feature: something a user can do or see afterwards that they couldn't before, built through every layer it needs, with one acceptance criterion and one test that proves it from the outside. Name it from the user's side ("init installs hooks for a Node subfolder"), never as a technical task ("add PrefixedFiles").
- Technical work (a new type, a refactor, a helper, a migration) goes inside the first feature that needs it, sized for that feature only. One exception: when a feature and the refactor it needs don't fit the size cap together, the refactor becomes its own `refactor` slice, directly before that feature. It changes no behaviour; its criterion is that the existing tests pass unchanged and the named feature then fits the cap. Never more than one refactor slice ahead of the feature it serves.
- A wide refactor (a rename or signature change across many call sites) that can't fit one refactor slice is cut expand-contract: one slice adds the new form beside the old, then one slice per batch of call sites moved over (batched by blast radius, each green on its own), then one slice removes the old form. Each is blocked by the one before.
- Work that serves no feature is a chore: scaffolding a new code base, a dependency bump, a CI or tooling fix. Mark it `chore` in the plan, with one criterion a check can prove. Scaffolding is a walking skeleton: the smallest code base that builds, runs one trivial path end to end and passes CI, nothing more; the first feature grows from it.
- Never slice by layer (types, then templates, then engine, then commands). A layer on its own ships nothing and can't be tested from outside.
- Start with the thinnest slice that runs end to end, often read-only or one narrow case ("doctor lists subfolder projects" before "init installs into them"). Each later slice widens it: another case, stack or command.
- Size: about 200 hand-written lines (outside generated files: goldens, lockfiles, snapshots) across at most 8 files, unless the repo's instructions set another cap. A bigger slice gets cut again, by case or by command, not by layer.
- Each slice names the files it expects to touch and the one test it adds. If you can't name them after reading the code, the slice isn't ready: back to step 1.
- A contract or sensitive change (matches `.claude/risk-paths.md`) is its own slice.
- If you can't write a slice's acceptance criterion as something a test checks, the requirement is still unclear: back to step 1.

## 3. Show the plan
One table: order, slice, acceptance criterion, test, files it touches, depends on. Then the questions still open and the slices they block. Say where a cut was hard and why. Wait for the user's ok.

## 4. Create the issues
- Write the agreed design where every issue can link it (the repo's docs or the tracker), so implementers don't re-decide it.
- One issue per slice: what changes for the user, the acceptance criterion, the test that proves it, the files it expects to touch, out of scope (name the later slices), blocking links.
- End with the issue ids in order and `/ship-wave <ids>`.
