# 00 — Overview

## Vision

CleanPress is a self-hosted, single-site content management platform that
delivers the features publishers actually use in WordPress — a block-based
editor, media library, users/roles, comments, menus, themes, and an
extensibility model — on a modern, statically-typed, cloud-portable stack:

- **Front office** (public site): React + TypeScript, server-rendered for SEO.
- **Back office** (admin dashboard): React + TypeScript SPA.
- **Backend**: .NET (C#), hosting every API and all server-side logic —
  content, media processing, search, auth, extensibility, jobs.
- **Database**: PostgreSQL as the single system of record.
- **Hosting**: Kubernetes, cloud-agnostic — deployable to any cloud or
  on-prem cluster without rewrites.

## Goals

1. **Content authoring parity with WordPress core** for posts, pages, custom
   post types, taxonomies, media, comments, menus, and a block editor.
2. **Operator parity**: roles/capabilities, revisions, drafts/scheduling,
   trash, site settings, a Site Health-style status page.
3. **Extensibility** as a first-class concern, not bolted on: plugins and
   themes are designed in from Phase 0, even though the marketplace ships
   later (see [roadmap](../roadmap.md)).
4. **Operability**: the platform must be observable, horizontally scalable,
   backup/restorable, and deployable via GitOps to any Kubernetes cluster.
5. **Type-safe, contract-first integration** between backend and both
   frontends — no hand-maintained duplicate types.

## Non-goals (for now)

These are explicitly deferred — see [`docs/roadmap.md`](../roadmap.md) and
[`docs/feature-matrix.md`](../feature-matrix.md) for when/if they're picked up:

- Multi-tenant SaaS (hosting many independent sites from one deployment) —
  see [ADR-0003](../adr/0003-single-site-not-multi-tenant.md). The
  architecture avoids decisions that would *block* this later, but does not
  build it now.
- A public plugin/theme marketplace with third-party billing.
- Full e-commerce (WooCommerce-equivalent) — sequenced as a later roadmap
  phase, built as an optional module against the same extensibility points
  ordinary plugins use.
- WordPress Multisite (network of sites under one install).
- Arbitrary third-party PHP-style plugin code execution. Extensibility is
  designed around defined hook points and sandboxed extension runtimes
  (see [ADR](../adr) list and [06-auth-and-security.md](06-auth-and-security.md)),
  not arbitrary server code upload.

## Guiding principles

- **Boring where it doesn't matter, opinionated where it does.** PostgreSQL,
  REST, Kubernetes, GitHub Actions — proven tooling. The genuine design
  effort goes into the domain model, the extensibility points, and the
  module boundaries.
- **Modular monolith first, split later.** One deployable backend with
  strict internal module boundaries (own schema, own migrations, mediated
  cross-module calls) so any module can be extracted into its own service
  later without a rewrite — see [ADR-0004](../adr/0004-modular-monolith-backend.md).
- **Cloud portability over cloud-native convenience.** Prefer Kubernetes
  operators (Postgres, Redis, object storage) that run identically on any
  cluster over managed cloud services, and put any unavoidable
  cloud-specific integration behind an interface. See
  [ADR-0001](../adr/0001-cloud-agnostic-kubernetes-hosting.md).
- **Contract-first APIs.** The backend's OpenAPI spec is the source of
  truth; both frontends consume a generated, typed client. No manually
  duplicated DTOs.
- **Everything as code.** Infrastructure (Terraform + Helm), database
  schema (EF Core migrations), and even content-type/taxonomy definitions
  are versioned and reviewable.

## How to read this plan

1. Start with [01-system-architecture.md](01-system-architecture.md) for the
   component map.
2. [02-domain-model.md](02-domain-model.md) defines the entities every other
   document assumes.
3. Documents 03–09 go deep on one concern each (database, API, frontend,
   security, infra, observability, testing).
4. [`docs/adr/`](../adr) records *why* — each significant, hard-to-reverse
   decision gets its own ADR, referenced from the relevant architecture doc.
5. [`docs/roadmap.md`](../roadmap.md) sequences the work into phases.
6. [`docs/feature-matrix.md`](../feature-matrix.md) tracks WordPress feature
   parity, phase by phase.
