# Risk paths template

A repo may keep its own list at `.claude/risk-paths.md`, written by hand or by a tool. A PR that touches a listed path starts at that tier (the floor); the gate may raise it, never lower it. When a repo has none, the gate uses this template: its generic globs work in any repo and the example globs match nothing.

```markdown
# Risk paths (read by the pr-gate skill)

One glob per bullet, in backticks, gitignore style (`**` crosses directories). A note may follow after ` - `. A file takes the highest tier among the globs it matches. The floor for a PR is the highest tier of any file it touches. A file matching no glob is medium. Low applies only when every changed file matches a low glob. The gate may raise a tier, never lower it.

## high
- `src/auth/**` - example: sign-in, tokens, sessions, permissions; replace with this repo's paths
- `src/storage/**` - example: where user data is written, read and deleted
- `src/api/**` - example: endpoints that scope data per user or project
- `src/scoring/**` - example: calculation formulas and their default parameters
- `.github/workflows/**`
- `.github/CODEOWNERS`
- `Dockerfile*`
- `docker-compose*`
- `infra/**` - Terraform, Pulumi, Kubernetes, Helm
- `**/*.tf`
- `pyproject.toml`
- `uv.lock`
- `package.json`
- `package-lock.json`
- `pnpm-lock.yaml`
- `openapi*.{yaml,yml,json}` - API contract
- `**/*.proto`
- `migrations/**` - stored data shape
- `.importlinter` - architecture rules
- `.dependency-cruiser.*` - architecture rules
- `.pre-commit-config.yaml` - the checks themselves
- `pytest.ini`
- `.coveragerc`
- `vitest.config.*`
- `jest.config.*`
- `.semgrep*`
- `.gitleaks.toml`
- `CLAUDE.md`
- `AGENTS.md`
- `.claude/**` - the rules agents follow, including this file

## needs-adr
- `openapi*.{yaml,yml,json}`
- `**/*.proto`
- `migrations/**`
- `.importlinter`
- `.dependency-cruiser.*`

## medium
- everything not listed elsewhere; add a glob here only to pull a path up from low

## low
- `docs/**`
- `**/*.md`
- `tests/**`
- `**/*.test.*`
- `**/*.spec.*`
```

Generic fallback when a repo has no list: auth, storage, isolation, CI, container/deploy, dependency manifests and lockfiles, and agent rule files are high. Docs-only and tests-only changes are low. Everything else is medium.
