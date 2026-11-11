# Gate comment template

Write it in plain words for a reader who is not a programmer. Keep it short: what changed, how sure we are, and what to look at.

```markdown
<!-- pr-gate -->
## Risk: **<low | medium | high>**

**Why this tier:** <one or two sentences: what the change touches; which risk-path rule set the floor, or why it was raised>

**Floor:** <tier>, from <`.claude/risk-paths.md` / the skill's default list>; needs ADR <yes / no>; CI weakened <no findings / n findings, listed under "For you to look at">

**Follow-ups filed:** <issue links for review findings outside this PR's scope, or "none">

**Evidence** (head <short sha>)
| Check | Result |
|---|---|
| GitHub checks | <all green: names> |
| `<command>` | exit 0 - <961 passed> |
| ... | ... |
| Fresh-context review | <n blocking (all fixed / answered), n nits> |
| Security review | <no findings / finding + resolution> |
| Screens | <links to before/after screenshots, or "no visible change"> |

**What the checks would miss:** <the honest gap: e.g. "how the new layout feels on a real phone", "whether 30 % is the right engineering limit">

**How to revert:** <e.g. "revert the squash commit; no stored data changes" or the extra step if data/config changed>

**For you to look at:** <medium: 1-3 screenshots and what to judge in each; high: the review summary and the one decision only a person can make; low: "nothing: the agent merges this">. <medium and high: the look-here permalinks, one per line, matching the `## Look here` section at the top of the PR body>

**Stacked on:** <#parent, or "none">

**Gate ran as:** <`<machine user>` (verified) / the user's own account (AGENT_GITHUB_LOGIN not set) / mismatch: <detail>>. <on a GitHub Free private repo add: "GitHub does not enforce these tiers here; this gate does.">
```
