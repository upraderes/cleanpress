---
name: cleanpress-dotnet-conventions
description: Use this skill when writing or reviewing any code under backend/ in the CleanPress repository — adding an endpoint, a module, an entity, a migration, a background job, or a test. Covers the modular-monolith rules, EF Core/PostgreSQL conventions, and API conventions specific to this codebase. Load cleanpress-architecture first if the change also involves a design decision, not just implementation within an already-decided pattern.
---

# CleanPress .NET backend conventions

These conventions implement the decisions in
`docs/architecture/02-domain-model.md`, `03-database-design.md`, and
`04-api-design.md` — read this skill as "how to write the code," and those
docs as "why it's structured this way."

## Module structure

Each module (`Content`, `Media`, `Identity`, `Comments`, `Navigation`,
`Appearance`, `Extensibility`, `Settings`, `Search`, `Audit`) is a
self-contained folder under `backend/src/Modules/{ModuleName}/` with:

```
{ModuleName}/
  Domain/           entities, value objects — no EF Core attributes here
  Application/      MediatR command/query handlers, validators
  Infrastructure/   EF Core DbContext, configurations, migrations
  Api/              minimal API endpoint definitions for this module
  {ModuleName}Module.cs   registers this module's services/endpoints
```

**Hard rule**: a module's code may reference another module only through
that module's `Application` layer public request/response types (dispatched
via MediatR), or by subscribing to its published domain events. Never
reference another module's `Domain` entities, `Infrastructure.DbContext`,
or EF Core types directly. This is enforced by NetArchTest rules in
`backend/tests/ArchitectureTests` — a new module dependency must pass
those before it's not "just a compile error away from being caught."

## Database conventions

- One `DbContext` per module, mapped to that module's own Postgres schema
  (`content`, `media`, `identity`, …) — set `HasDefaultSchema` on the
  context, never mix schemas in one context.
- `snake_case` naming via `UseSnakeCaseNamingConvention()`.
- Primary keys: `Guid` (UUIDv7 via `Guid.CreateVersion7()` or equivalent),
  never `int`/`identity` columns.
- Timestamps: `DateTimeOffset`, stored as `timestamptz`, always UTC at the
  application boundary.
- Flexible fields (post meta, settings, plugin/theme settings): a `jsonb`
  column via `.HasColumnType("jsonb")`, with a GIN index added in the
  migration when the field is queried (not just stored).
- Every mutable aggregate root gets an EF Core concurrency token mapped to
  Postgres `xmin` (`.IsRowVersion()` on a `uint` shadow property) — don't
  add a manual `RowVersion` column unless there's a specific reason
  `xmin` doesn't work.
- New migrations go in that module's own migrations folder and history
  table (`__EFMigrationsHistory_{module}`) — never a shared migrations
  history across modules.
- Follow the expand/contract pattern for any change that isn't purely
  additive (add nullable/new column → backfill in a later
  release → drop old column in a release after that only once nothing
  reads it) — see `docs/architecture/03-database-design.md#migrations`.
  A migration that isn't safe to apply before the new code deploys is
  structured wrong.

## API conventions

- Minimal APIs, grouped per module, mounted under `/api/v1/...`.
- Resource-oriented routes; non-CRUD actions are POST sub-resources
  (`POST /api/v1/posts/{id}/publish`), not verb-suffixed RPC endpoints.
- Every endpoint that mutates data has an explicit `[Authorize(Policy =
  "...")]` capability policy — never an endpoint that relies solely on
  "the frontend won't show the button."
- Validation via FluentValidation; validation failures map to RFC 7807
  Problem Details automatically through the shared error-handling
  middleware — don't hand-roll a different error shape.
- New/changed endpoints must keep the OpenAPI spec (Swashbuckle-generated)
  additive within `v1` unless this is a deliberate, documented breaking
  change (new `v2` route) — see `docs/architecture/04-api-design.md#versioning-policy`.
- Cross-module side effects go through a MediatR domain event
  (`INotification`), published after the triggering transaction commits
  (outbox pattern for anything that also needs to survive a crash between
  commit and publish, e.g. webhook delivery triggers).

## Testing

- Unit tests: xUnit + FluentAssertions, one test project per module,
  mocked dependencies, no real I/O.
- Integration tests: Testcontainers-provisioned real Postgres/Redis — for
  anything touching a `DbContext`, a real migration, or a `jsonb` query.
- New cross-module dependencies must pass the architecture tests, not just
  compile.
- See `docs/architecture/09-testing-strategy.md` for what's required to
  merge vs. what only gates promotion to staging.

## When in doubt

If a change doesn't fit an existing module cleanly, or seems to need a new
cross-module interaction pattern not covered here, check
`docs/architecture/02-domain-model.md` and the `cleanpress-architecture`
skill before improvising — and if it's genuinely a new pattern, that's an
ADR (`docs/adr/`), not a silent precedent.
