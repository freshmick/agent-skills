# Risk paths template

A repo may keep its own list at `.claude/risk-paths.md`, written by hand or by a tool. It sets the repo's lane setting and the paths that decide each PR's lane and review depth. When a repo has none, the gate uses this template with `lane: live`: its generic globs work in any repo and the example globs match nothing.

```markdown
# Risk paths (read by the pr-gate skill)

lane: live

`lane: mvp` (no real users): the agent merges every PR except those touching a sensitive path or marked `Door: one-way`. `lane: live` (real users, or someone else's repo): the agent merges docs, tests and regenerated files only. No line means `live`.

One glob per bullet, in backticks, gitignore style (`**` crosses directories). A note may follow after ` - `. A file matching a sensitive glob is sensitive, even if it also matches a docs-and-tests glob. A PR is docs-and-tests only when every changed file matches a docs-and-tests glob. The gate may move a PR to the person's lane, never to the agent lane. Older files name these sections `## high` and `## low`; the gate reads them the same way.

## sensitive
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

## docs and tests
- `docs/**`
- `**/*.md`
- `tests/**`
- `**/*.test.*`
- `**/*.spec.*`
```

Generic fallback when a repo has no list: auth, storage, isolation, CI, container/deploy, dependency manifests and lockfiles, and agent rule files are sensitive. Docs-only and tests-only changes are docs and tests. Everything else is ordinary, and in a `live` repo it still goes to the person.
