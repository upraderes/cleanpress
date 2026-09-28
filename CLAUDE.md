# CleanPress

A WordPress-equivalent CMS: React+TypeScript front office (Next.js) and
back office (Vite SPA), a .NET modular-monolith backend, PostgreSQL, hosted
cloud-agnostically on Kubernetes.

**This repository currently contains the architecture plan, not yet
application code.** See `docs/roadmap.md` for the phase in which each part
gets built, and don't scaffold ahead of the current phase without checking
with the user first.

## Before making any non-trivial change

Use the `cleanpress-architecture` skill (and the more specific
`cleanpress-dotnet-conventions`, `cleanpress-react-conventions`, or
`cleanpress-infra` skill depending on which part of the repo you're
touching) — they point to the authoritative docs and encode this
project's conventions instead of leaving them to be rediscovered.

## Map

| Path | What |
|---|---|
| `docs/architecture/` | The architecture plan (00–09), read in order |
| `docs/adr/` | Why each hard-to-reverse decision was made |
| `docs/roadmap.md` | Phased delivery plan |
| `docs/feature-matrix.md` | WordPress feature parity tracking |
| `backend/` | .NET modular monolith (not yet scaffolded — see roadmap Phase 0) |
| `frontend-public/` | Next.js front office (not yet scaffolded) |
| `frontend-admin/` | React + Vite back office (not yet scaffolded) |
| `packages/` | Shared TS packages: api-client, ui-kit, config (not yet scaffolded) |
| `infra/` | Helm charts, Terraform modules (not yet scaffolded) |
| `.claude/skills/` | Project-specific Claude Code skills implementing the above |

## Key decisions (see the linked ADR for full rationale)

- Cloud-agnostic Kubernetes hosting, not a single cloud's PaaS — [ADR-0001](docs/adr/0001-cloud-agnostic-kubernetes-hosting.md)
- PostgreSQL as the sole system of record — [ADR-0002](docs/adr/0002-postgresql-as-primary-datastore.md)
- Single-site CMS, not multi-tenant SaaS — [ADR-0003](docs/adr/0003-single-site-not-multi-tenant.md)
- Modular monolith backend, not microservices — [ADR-0004](docs/adr/0004-modular-monolith-backend.md)
- Next.js (public) + Vite SPA (admin), not one shared frontend app — [ADR-0005](docs/adr/0005-nextjs-public-vite-admin-split.md)
- Plugin extensibility via webhooks + WASM sandbox, not arbitrary code — [ADR-0006](docs/adr/0006-plugin-sandboxing-not-arbitrary-code.md)
- OpenIddict self-hosted identity, not a managed cloud IdP — [ADR-0007](docs/adr/0007-openiddict-self-hosted-identity.md)

Don't contradict an accepted ADR in new code without flagging it to the
user first — these are decisions, not suggestions.
