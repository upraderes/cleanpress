# CleanPress

CleanPress is a from-scratch, WordPress-equivalent content management platform:
a **React + TypeScript front office** (the public website), a **React +
TypeScript back office** (the admin dashboard), and a **.NET backend** that
hosts every API and all server-side intelligence, backed by **PostgreSQL**.

This repository currently holds the **architecture plan** and the **Claude
Code skills** used to build it. No application code has been scaffolded yet —
see [`docs/roadmap.md`](docs/roadmap.md) for the phase in which each part of
the system gets built.

## Start here

| Document | What it covers |
|---|---|
| [`docs/architecture/00-overview.md`](docs/architecture/00-overview.md) | Vision, goals, non-goals, guiding principles |
| [`docs/architecture/01-system-architecture.md`](docs/architecture/01-system-architecture.md) | High-level component diagram and request flow |
| [`docs/architecture/02-domain-model.md`](docs/architecture/02-domain-model.md) | Modules and entities (Content, Media, Identity, Comments, …) |
| [`docs/architecture/03-database-design.md`](docs/architecture/03-database-design.md) | PostgreSQL schema strategy |
| [`docs/architecture/04-api-design.md`](docs/architecture/04-api-design.md) | REST conventions, versioning, webhooks |
| [`docs/architecture/05-frontend-architecture.md`](docs/architecture/05-frontend-architecture.md) | Front office (Next.js) and back office (SPA) design |
| [`docs/architecture/06-auth-and-security.md`](docs/architecture/06-auth-and-security.md) | AuthN/AuthZ, roles/capabilities, security baseline |
| [`docs/architecture/07-infrastructure-and-deployment.md`](docs/architecture/07-infrastructure-and-deployment.md) | Kubernetes, CI/CD, IaC |
| [`docs/architecture/08-observability.md`](docs/architecture/08-observability.md) | Metrics, logs, traces, SLOs |
| [`docs/architecture/09-testing-strategy.md`](docs/architecture/09-testing-strategy.md) | Test pyramid and tooling |
| [`docs/adr/`](docs/adr) | Architecture Decision Records |
| [`docs/roadmap.md`](docs/roadmap.md) | Phased delivery plan |
| [`docs/feature-matrix.md`](docs/feature-matrix.md) | WordPress feature parity tracking |

## Repository layout (planned)

```
backend/            .NET modular monolith — all APIs and server-side logic
frontend-public/    Next.js front office (public website, SSR/ISR)
frontend-admin/     React + Vite back office (admin dashboard)
packages/           Shared TS packages (API client, UI kit, config)
infra/              Helm charts, Terraform modules, k8s manifests
docs/                Architecture plan, ADRs, roadmap
.claude/skills/     Project-specific Claude Code skills
```

## Key decisions at a glance

- **Hosting:** Cloud-agnostic Kubernetes (see [ADR-0001](docs/adr/0001-cloud-agnostic-kubernetes-hosting.md))
- **Database:** PostgreSQL as the sole system of record (see [ADR-0002](docs/adr/0002-postgresql-as-primary-datastore.md))
- **Scope:** Single-site CMS, not multi-tenant SaaS (see [ADR-0003](docs/adr/0003-single-site-not-multi-tenant.md))
- **Backend style:** Modular monolith, not microservices, for MVP (see [ADR-0004](docs/adr/0004-modular-monolith-backend.md))
