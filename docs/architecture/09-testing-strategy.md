# 09 — Testing Strategy

## Backend (`backend/`)

- **Unit tests**: xUnit + FluentAssertions, one test project per module,
  fast, no I/O — pure domain logic and handler behavior with mocked
  dependencies.
- **Integration tests**: Testcontainers spins up real Postgres and Redis
  per test run — no mocked database. Exercises a module's `DbContext`,
  migrations, and repository/query handlers against the real engine,
  catching the class of bug unit tests with an in-memory provider miss
  entirely (JSONB queries, concurrency tokens, constraint violations).
- **Architecture tests**: NetArchTest rules enforcing the module-boundary
  rules from [02-domain-model.md](02-domain-model.md#cross-module-interaction-rules)
  in CI — a module referencing another module's `DbContext` or entity
  types directly fails the build, not just code review.
- **Contract tests**: the generated OpenAPI spec is validated against a
  running instance (Schemathesis or a custom Dredd-style check) on every
  PR that touches an endpoint — the spec can't silently drift from the
  implementation.
- **API-level tests**: `WebApplicationFactory`-based tests hitting real
  HTTP endpoints in-process, covering authorization policies (a
  Contributor calling a `manage_options` endpoint must get 403, not 500)
  end to end through the actual middleware pipeline.

## Frontend (`frontend-public/`, `frontend-admin/`, `packages/`)

- **Unit/component tests**: Vitest + React Testing Library — components,
  hooks, and the block-editor's block registration logic in isolation.
- **End-to-end tests**: Playwright, run against a full environment (either
  the local docker-compose stack or a PR preview environment — see
  [07-infrastructure-and-deployment.md](07-infrastructure-and-deployment.md)).
  Critical flows covered from Phase 1 onward and never allowed to regress:
  - Create, edit, and publish a post (back office) → appears correctly on
    the public site (front office), including revalidation.
  - Upload media, insert into a post, verify renditions render.
  - Create a user with a given role, verify capability-gated UI and API
    responses match (an Author can't see "manage plugins").
  - Submit and moderate a public comment.
  - Install/activate/deactivate a plugin against the webhook-based
    extensibility surface (from Phase 4 — see [roadmap](../roadmap.md)).
- **Visual regression**: Playwright screenshot comparison (or Chromatic if
  budget allows) on the shared `ui-kit` components and on each theme
  package, catching unintended visual drift on shared design tokens.
- **Performance budget**: Lighthouse CI against `frontend-public` on every
  PR touching it — fails the build on regression past the budget set in
  [08-observability.md](08-observability.md#slos-initial-targets-revisited-once-theres-real-traffic).

## Cross-cutting

- **Load testing**: k6 scripts against the API's hottest paths (public
  post read, publish, search) — run on demand pre-launch and after major
  changes to the caching/read-replica path, not on every PR.
- **Security testing**: SAST (CodeQL) and dependency-vulnerability
  scanning (Dependabot/Renovate, `dotnet list package --vulnerable`,
  `npm audit`) on every PR — see
  [06-auth-and-security.md](06-auth-and-security.md#transport-and-infrastructure-security).
  A periodic (not per-PR) authenticated DAST pass and, pre-launch, a manual
  penetration test.
- **Migration/import fixtures**: a canonical sample WordPress WXR export
  file checked into `backend/tests/fixtures/`, used both to test the
  WordPress-import tooling (see [roadmap](../roadmap.md)) and as a
  realistic seed dataset for local development and Playwright e2e runs.

## What blocks a merge

CI must pass: lint + typecheck (both frontends), unit + integration +
architecture tests (backend), unit + component tests (frontend), and, for
PRs touching the public site or a shared UI package, Lighthouse budget and
the Playwright critical-flow suite. Full e2e suite and load tests are not
required per-PR (too slow) but run on every merge to `main` and block
promotion to staging on failure.
