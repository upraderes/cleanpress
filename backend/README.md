# backend

The .NET modular-monolith API. Not yet scaffolded — see
[`docs/roadmap.md`](../docs/roadmap.md) (Phase 0) for when this happens.

When it is, this directory will hold one .NET solution with a module per
domain concern (`src/Modules/Content`, `Media`, `Identity`, `Comments`,
`Navigation`, `Appearance`, `Extensibility`, `Settings`, `Search`, `Audit`),
each owning its own PostgreSQL schema and EF Core `DbContext`.

Read before writing any code here:
- [`docs/architecture/02-domain-model.md`](../docs/architecture/02-domain-model.md)
- [`docs/architecture/03-database-design.md`](../docs/architecture/03-database-design.md)
- [`docs/architecture/04-api-design.md`](../docs/architecture/04-api-design.md)
- The `cleanpress-dotnet-conventions` Claude Code skill (`.claude/skills/`)
