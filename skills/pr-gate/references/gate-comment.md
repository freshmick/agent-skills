# Gate comment template

Plain words, for a reader who is not a programmer. Short: a small change gets a few lines, never a page. One line per field; leave a field out when it has nothing to say.

```markdown
<!-- pr-gate -->
**Risk: <low | medium | high>** (head <short sha>)

**Why:** <one sentence: what the change touches and which rule set the tier>
**Checks:** <GitHub checks green; `pytest` 81 passed; pre-commit clean>
**Reviews:** <fresh-context: n blocking, fixed; security: no findings; /code-review high: ...>
**Misses:** <the honest gap the checks don't cover, one line>
**Revert:** <"revert the squash commit", or the extra step>
**Follow-ups:** <issue links, if any>
**Look at:** <medium/high: the 1-3 decisions with permalinks, or "nothing to decide"; low: omit>
<only when true: "CI weakened: file:line ..." / "needs ADR" / "stacked on #M" / identity mismatch / "GitHub does not enforce these tiers here">
```
