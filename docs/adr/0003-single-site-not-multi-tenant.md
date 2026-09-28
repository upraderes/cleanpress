# ADR-0003: Single-site CMS, not multi-tenant SaaS

## Status
Accepted

## Context
WordPress ships in two very different shapes: a self-hosted, single-site
install (wordpress.org) and a multi-tenant SaaS hosting many independent
sites (wordpress.com). These imply materially different architectures —
tenant isolation, per-tenant data partitioning, per-tenant theming/domains,
usage-based billing — that are expensive to retrofit if assumed away
early, and expensive to build speculatively if never needed.

## Decision
Build CleanPress as a **single-site CMS**: one deployment serves one
website, matching classic self-hosted WordPress. Multi-tenancy is
explicitly **not** built now, but the architecture avoids decisions that
would make adding it later a rewrite rather than an extension — see
[00-overview.md](../architecture/00-overview.md#non-goals-for-now).

## Consequences

**Positive**
- Materially simpler data model: no `tenant_id` scoping on every table, no
  per-tenant connection routing, no cross-tenant isolation testing burden.
- Simpler auth model: one Identity realm per deployment instead of
  per-tenant realms/domains.
- Faster path to a usable MVP — the full core CMS feature set (see
  [feature-matrix.md](../feature-matrix.md)) is deliverable without first
  solving multi-tenancy.

**Negative / trade-offs**
- If a multi-tenant SaaS offering is wanted later, it is a genuine
  architecture project, not a config flag: it would likely mean promoting
  the Identity module to its own deployable (already anticipated — see
  [06-auth-and-security.md](../architecture/06-auth-and-security.md#identity-provider)),
  adding tenant scoping to every module's schema, and rethinking the
  single-active-theme assumption in
  [05-frontend-architecture.md](../architecture/05-frontend-architecture.md#theming-mvp).
- Deploying CleanPress for multiple independent sites today means multiple
  independent deployments (separate namespaces/clusters), not one
  deployment serving many — acceptable at small scale, operationally
  heavier at large scale than true multi-tenancy would be.

## Forward-compatibility choices made anyway
To keep the door open without paying the cost now:
- UUIDs (not sequential integers) as primary keys everywhere, so records
  from different future tenants/deployments never collide on merge/import.
- The modular-monolith module boundaries (see
  [02-domain-model.md](../architecture/02-domain-model.md)) mean a
  `tenant_id` column, if added later, is added schema-by-schema, not as a
  single sweeping cross-cutting change.
- Object storage keys are already namespaced by deployment, not assumed
  global.

## Alternatives considered
- **Multi-tenant from day one**: rejected — the stakeholder confirmed
  single-site scope, and building tenant isolation, per-tenant
  provisioning, and billing before a single core CMS feature exists would
  significantly delay reaching a usable product.
- **Multi-tenant via separate deployment-per-tenant only (no shared
  platform)**: this is effectively what single-site + "deploy it again for
  another site" already gives; no additional decision needed.
