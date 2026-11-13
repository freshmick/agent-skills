---
name: ship-grill
description: Matt Pocock's grill-me, with one line added for the ship-wave flow - it doesn't ask how to split the work into PRs, since /to-tickets does that. Pass `docs` for his grill-with-docs instead (decision records and a glossary). Use when the user says "/ship-grill" or "/ship-grill docs".
license: MIT
argument-hint: "[docs]"
---

# Ship grill

This is Matt Pocock's `grill-me` and `grill-with-docs` from [mattpocock/skills](https://github.com/mattpocock/skills). The interview is all his `grilling` skill; ship-wave only adds the rule below.

Call the Skill tool with "grilling". When `$ARGUMENTS` is `docs`, also call it with "domain-modeling".

The one rule ship-wave adds: don't ask how to split the work into PRs or issues, or in what order to ship it. After the interview, `/to-spec` writes the design down and `/to-tickets` cuts it into issues.
