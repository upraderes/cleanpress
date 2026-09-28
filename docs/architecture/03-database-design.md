# 03 — Database Design

PostgreSQL is the single system of record (see
[ADR-0002](../adr/0002-postgresql-as-primary-datastore.md)). This document
covers schema strategy, indexing, search, replication, and backups.

## Schema-per-module

Each backend module (see [02-domain-model.md](02-domain-model.md)) owns a
dedicated Postgres **schema** in the same database/cluster:
`content`, `media`, `identity`, `comments`, `navigation`, `appearance`,
`extensibility`, `settings`, `search`, `audit`.

- Each module has its own EF Core `DbContext` and its own migrations
  directory (`backend/src/Modules/{Module}/Migrations`).
- No foreign keys across schemas at the database level — cross-module
  references are stored as plain UUIDs and validated in application code,
  keeping modules independently migratable and, eventually, extractable
  into separate services/databases without a data-model rewrite.
- One physical database for MVP (simpler ops, still logically separated);
  nothing prevents pointing a module's `DbContext` at a different
  connection string later if a module needs to scale independently.

## Naming and types

- `snake_case` table/column names (EF Core `UseSnakeCaseNamingConvention`),
  matching Postgres convention and keeping raw SQL/psql sessions readable.
- Primary keys: UUID v7 (time-ordered) — sortable like an autoincrement id
  without a central sequence, safe for eventual multi-primary scenarios.
- Timestamps: `timestamptz`, always UTC.
- Flexible/custom fields (`PostMeta`, `CommentMeta`, `Option`,
  `ThemeSetting`, `PluginSetting`): `jsonb` column, with a **GIN index**
  when queried by key/value, and a JSON Schema validated in the
  application layer for known keys (see
  [02-domain-model.md](02-domain-model.md#settings-schema-settings)).
- Optimistic concurrency: a `xmin`-based EF Core concurrency token on
  mutable aggregates (Post, User, Option) to catch concurrent-edit
  conflicts — surfaced in the back office as "this was changed by someone
  else" (see [05-frontend-architecture.md](05-frontend-architecture.md)).

## Content lifecycle: draft → published → trash

Mirrors WordPress's status model directly on `Post.status`
(`draft | pending_review | scheduled | published | trash`), rather than a
separate table, so a single indexed column drives all list/filter queries.
Trashed content is soft-deleted with a `trashed_at` timestamp and purged by
a scheduled job after a retention window (default 30 days, configurable via
`Settings.Option`).

## Full-text search

- **MVP (Phases 1–4):** Postgres native full-text search. A generated
  `tsvector` column on the `search.search_document` projection (title +
  body + excerpt + taxonomy terms, weighted: title > excerpt > body),
  backed by a GIN index. No extra infrastructure.
- **Scale trigger (Phase 5+):** if query latency or relevance needs outgrow
  Postgres FTS (faceted search, typo tolerance, large catalogs), the same
  `search.search_document` projection is shipped to OpenSearch/Elasticsearch
  via the existing domain-event pipeline — the write side doesn't change,
  only where the read side queries. See [roadmap](../roadmap.md).

## Read/write split

- All writes go to the **primary**.
- The **public front office** reads from a **streaming replica** —
  read-heavy, latency-tolerant-by-a-few-hundred-ms traffic that shouldn't
  compete with admin writes.
- The **back office** always reads from the **primary** (editors need
  read-your-writes consistency; replica lag would show stale content in
  the editor).
- Run via the CloudNativePG operator, which manages replication, automated
  failover, and connection routing (a read-write and a read-only service
  endpoint) natively on Kubernetes — see
  [07-infrastructure-and-deployment.md](07-infrastructure-and-deployment.md).

## Connection pooling

PgBouncer (transaction pooling mode) in front of Postgres — the API runs as
multiple replicas in Kubernetes and can otherwise exhaust Postgres's
connection limit under load. Configured per-service (API vs. workers) with
separate pool sizes.

## Partitioning

Time-based range partitioning (native Postgres declarative partitioning) on
high-growth, append-mostly tables once volume warrants it — primarily
`audit.audit_log_entry` and `comments.comment`. Not needed at MVP scale;
called out here so the schema (`created_at` as a leading, non-nullable,
indexed column) doesn't have to change when it is.

## Migrations

- EF Core Migrations, one migration history per module/schema
  (`__EFMigrationsHistory_content`, `__EFMigrationsHistory_media`, …).
- Applied by a dedicated Kubernetes **Job** that runs before a new API
  version's pods are rolled out (CI/CD gate — see
  [07-infrastructure-and-deployment.md](07-infrastructure-and-deployment.md)),
  never by the application on startup, so migrations are observable,
  ordered, and don't race across replicas.
- Every migration must be backward-compatible with the *previous* API
  version for the duration of a rolling deploy (expand/contract pattern:
  add nullable columns first, backfill, then drop old columns in a later
  release).

## Backups and recovery

- Continuous WAL archiving to object storage (via CloudNativePG's built-in
  Barman Cloud integration) + nightly base backups → point-in-time recovery.
- Velero for whole-namespace/volume disaster recovery (cluster migration,
  environment cloning).
- Backup restore is tested on a schedule (Phase 5, see
  [roadmap](../roadmap.md)), not just configured and forgotten.

## Why not MongoDB or Cosmos DB

Covered in full in
[ADR-0002](../adr/0002-postgresql-as-primary-datastore.md); in short: the
domain is fundamentally relational (users↔roles↔capabilities, posts↔terms,
multi-entity transactions like "publish" touching Post + PostMeta + Term
associations atomically), and JSONB gives the same per-post flexible-field
story WordPress's `wp_postmeta` provides, without giving up transactions,
foreign keys, or a mature migration story.
