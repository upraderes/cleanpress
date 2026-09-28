---
name: cleanpress-architecture
description: Use this skill whenever working in the CleanPress repository and a change touches system design — adding/moving a module, changing a data model, adding an API endpoint, choosing a library, changing how something is hosted/deployed, or any decision with more than one reasonable approach. Also use it when asked "how should X be built" or "where does Y belong" in this repo. It points to the authoritative architecture plan and ADRs instead of re-deriving decisions from scratch, and tells you when a new ADR is required.
---

# CleanPress architecture

CleanPress is a WordPress-equivalent CMS: React+TypeScript front office
(Next.js) and back office (Vite SPA), a .NET modular-monolith backend, and
PostgreSQL, hosted cloud-agnostically on Kubernetes. Before making or
proposing a non-trivial design decision in this repo, ground it in the
existing plan rather than re-deriving it:

1. **Read the relevant architecture doc first**: `docs/architecture/`,
   numbered 00–09 (overview, system architecture, domain model, database,
   API, frontend, auth/security, infra/deployment, observability,
   testing). Find the doc that covers the area you're touching before
   writing code or proposing an approach.
2. **Check `docs/adr/` for a decision that already covers this.** Every
   hard-to-reverse choice (hosting model, database, tenancy, backend
   architecture style, frontend split, plugin sandboxing, identity
   provider) has a numbered ADR with its context, decision, and
   consequences. Don't contradict an accepted ADR without flagging it
   explicitly to the user — ADRs are decisions, not suggestions.
3. **Check `docs/roadmap.md` before building something "ahead of its
   phase."** If a feature is marked `Deferred` in
   `docs/feature-matrix.md` or scoped to a later phase in the roadmap,
   building it now is scope creep — confirm with the user first rather
   than silently expanding the phase you're in.
4. **Respect the module boundaries in `docs/architecture/02-domain-model.md`.**
   The backend is a modular monolith: each module (Content, Media,
   Identity, Comments, Navigation, Appearance, Extensibility, Settings,
   Search, Audit) owns its own schema and `DbContext`. A module must never
   directly query another module's tables — only mediated queries
   (MediatR) or domain events cross a module boundary. This is enforced by
   architecture tests in CI; don't write code that would fail them.
5. **When a new significant, hard-to-reverse decision comes up** (a new
   dependency that locks in a vendor, a change to the hosting model, a new
   cross-cutting pattern), write a new ADR in `docs/adr/`, numbered
   sequentially, following the existing format (Status / Context /
   Decision / Consequences / Alternatives considered). Link it from the
   relevant architecture doc, the same way existing ADRs are
   cross-linked.
6. **When you build something the plan describes**, update
   `docs/feature-matrix.md`'s Status column for that row in the same
   change — the matrix should reflect reality, not just intent.

## Quick reference: where things live

| Concern | Doc |
|---|---|
| Vision, goals, non-goals | `docs/architecture/00-overview.md` |
| Component map / request flow | `docs/architecture/01-system-architecture.md` |
| Entities, modules, module boundary rules | `docs/architecture/02-domain-model.md` |
| Postgres schema strategy, migrations | `docs/architecture/03-database-design.md` |
| REST conventions, webhooks/hooks | `docs/architecture/04-api-design.md` |
| Next.js / Vite frontend design | `docs/architecture/05-frontend-architecture.md` |
| AuthN/AuthZ, roles/capabilities, plugin sandboxing | `docs/architecture/06-auth-and-security.md` |
| Kubernetes, CI/CD, Terraform | `docs/architecture/07-infrastructure-and-deployment.md` |
| Metrics/logs/traces, SLOs | `docs/architecture/08-observability.md` |
| Test pyramid, what blocks a merge | `docs/architecture/09-testing-strategy.md` |
| Why a decision was made | `docs/adr/000N-*.md` |
| What's built when | `docs/roadmap.md`, `docs/feature-matrix.md` |

If a question can't be answered from these docs, say so explicitly rather
than guessing — architecture decisions in this repo are meant to be
traceable to a written rationale.
