# ADR-0002: PostgreSQL as the primary datastore

## Status
Accepted

## Context
The stakeholder offered a choice between PostgreSQL, MongoDB, and Cosmos
DB. A WordPress-equivalent CMS domain has a mix of strongly relational data
(users, roles, capabilities, taxonomies, post-term associations) and
flexible/schema-varying data (post meta, plugin settings, theme settings —
WordPress's `wp_postmeta`/`wp_options` pattern).

## Decision
Use **PostgreSQL** as the single system of record for all modules, using
`jsonb` columns (with GIN indexes) for the flexible-field use cases
instead of a document database. See
[03-database-design.md](../architecture/03-database-design.md).

## Consequences

**Positive**
- Transactional integrity across multi-entity operations that must be
  atomic — e.g. "publish a post" touches `Post`, `PostMeta`, and
  `PostTerm` rows together. A document database would need either
  denormalization (duplicated term data on the post document, with its own
  consistency problems) or application-level multi-document transaction
  handling.
- Relational modeling is the natural fit for RBAC (`User` ↔ `Role` ↔
  `Capability`) and taxonomies (`Term` ↔ `Taxonomy` ↔ `Post`) — exactly the
  parts of the WordPress domain that are most safety-critical to get
  right.
- `jsonb` + GIN indexes gives the same "arbitrary key/value per post"
  flexibility WordPress's `wp_postmeta` provides, without losing ACID
  transactions, foreign keys, or a single mature migration story (EF Core
  Migrations) across the whole schema.
- Mature EF Core provider, strong operational tooling (CloudNativePG for
  Kubernetes-native HA — see
  [ADR-0001](0001-cloud-agnostic-kubernetes-hosting.md)), native full-text
  search sufficient for MVP scale (see
  [03-database-design.md](../architecture/03-database-design.md#full-text-search)).
- Runs identically on every cloud and on-prem via CloudNativePG,
  reinforcing the cloud-agnostic decision rather than fighting it.

**Negative / trade-offs**
- Schema changes require migrations (mitigated by the expand/contract
  discipline in
  [03-database-design.md](../architecture/03-database-design.md#migrations));
  a document database would allow shape changes without a migration step,
  at the cost of no schema safety net at all.
- Horizontal write scaling is harder than a natively sharded document
  store; not a concern at single-site CMS scale, and Postgres's
  read-replica story (see
  [03-database-design.md](../architecture/03-database-design.md#readwrite-split))
  covers the actual bottleneck (read-heavy public traffic).

## Alternatives considered
- **MongoDB**: natural fit for varied content shapes, but weak
  multi-document transaction guarantees relative to Postgres, and no
  native foreign-key enforcement for the RBAC/taxonomy relationships that
  are core to this domain. Would also mean re-deriving a lot of what
  `jsonb` already gives Postgres.
- **Cosmos DB**: strong multi-region distribution and multi-model support,
  but it's an Azure-managed service — directly in tension with the
  cloud-agnostic hosting decision ([ADR-0001](0001-cloud-agnostic-kubernetes-hosting.md)),
  and materially more expensive at small scale than a self-hosted Postgres
  cluster.
- **Polyglot persistence** (Postgres for relational, Mongo for content):
  rejected for MVP — two datastores to operate, back up, and keep
  consistent, for a benefit ( `jsonb` already provides) that doesn't
  justify the operational cost at this stage. A dedicated search index
  (OpenSearch) is still added later, but as a *derived, rebuildable*
  projection, not a system of record — see
  [03-database-design.md](../architecture/03-database-design.md#full-text-search).
