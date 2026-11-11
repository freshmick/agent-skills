---
name: slice-work
description: Break an agreed design into vertical slices, one issue per slice, ready for ship-wave. Use when the user says "break out the work", "break this down", "create the issues" or "slice this", or when a design discussion ends and implementation is next.
---

# Slice work

Input: the design agreed in this conversation, or a design doc. Output: issues in the tracker, one vertical slice each, in order, ready for /ship-wave.

## 1. Find the slices
- A slice is a change that works and can be seen on its own (by a user, the CLI or CI): one acceptance criterion, one test that proves it, through every layer it needs.
- Never slice by layer (types, then templates, then engine, then commands). A layer on its own ships nothing and can't be tested from outside.
- Start with the thinnest slice that runs end to end, often read-only or one narrow case ("doctor lists subfolder projects" before "init installs into them"). Each later slice widens it: another case, stack or command.
- Size: about 300 changed lines outside generated files (goldens, lockfiles, snapshots). A bigger slice gets cut again, by case or by command, not by layer.
- A contract or sensitive change (matches `.claude/risk-paths.md`) is its own slice.
- A point the design left open is a question for the user, not a slice.

## 2. Show the plan
One table: order, slice, acceptance criterion, depends on, rough size. Then the open questions. Say where a cut was hard and why. Wait for the user's ok.

## 3. Create the issues
- Write the agreed design where every issue can link it (the repo's docs or the tracker), so implementers don't re-decide it.
- One issue per slice: what changes for the user, the acceptance criterion, the test that proves it, out of scope (name the later slices), blocking links.
- End with the issue ids in order and `/ship-wave <ids>`.
