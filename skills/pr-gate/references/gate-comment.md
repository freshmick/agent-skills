# Gate comment template

Plain words, for a reader who is not a programmer. Short: a small change gets a few lines, never a page. One line per field; leave a field out when it has nothing to say.

```markdown
<!-- pr-gate -->
**Lane: <agent | person's>** (repo setting `<mvp | live>`, head <short sha>)

**Why:** <one sentence: what the change touches and which rule set the lane>
**Checks:** <GitHub checks green; `pytest` 81 passed; pre-commit clean>
**Reviews:** <docs or tests only: diff read; ordinary: fresh-context, n blocking, fixed; sensitive: two reviewers, n blocking, fixed; second reviewer: no findings>
**Misses:** <the honest gap the checks don't cover, one line>
**Revert:** <"revert the squash commit", or the extra step>
**Look at:** <person's lane: the 1-3 decisions with permalinks, or "nothing to decide"; agent lane: omit>
<only when true: "CI weakened: file:line ..." / "needs ADR" / "stacked on #M" / identity mismatch / "GitHub does not enforce these lanes here">
```
