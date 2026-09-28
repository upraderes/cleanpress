# 02 — Domain Model

The backend is a **modular monolith**: one deployable, internally split into
modules with hard boundaries. Each module owns its own PostgreSQL schema, its
own EF Core `DbContext`, and its own migrations. Modules never query another
module's tables directly — cross-module reads go through an in-process
mediator (MediatR queries), and cross-module side effects go through domain
events. This is what makes it possible to extract a module into its own
service later (see [ADR-0004](../adr/0004-modular-monolith-backend.md))
without a rewrite.

## Modules and core entities

### Content (`schema: content`)
The heart of the system — WordPress's `wp_posts`/`wp_postmeta`/`wp_terms`
equivalent.

- **Post** — `id, post_type, status (draft/pending/published/scheduled/trash),
  title, slug, content (block JSON), excerpt, author_id, published_at,
  updated_at, parent_id (for pages/hierarchy)`. `post_type` is extensible
  (`post`, `page`, and custom post types registered by plugins).
- **PostMeta** — `post_id, key, value (jsonb)`. Flexible custom fields,
  indexed with a GIN index for query support (WordPress's `wp_postmeta`,
  but JSONB instead of serialized scalars).
- **Revision** — immutable snapshot of a Post at a point in time, for
  version history and autosave recovery.
- **Term / Taxonomy** — `Taxonomy` (e.g. `category`, `tag`, or a custom
  taxonomy) and `Term` (a value within a taxonomy), with a `PostTerm`
  join table.
- **CustomPostTypeDefinition** — registered post types (built-in or
  plugin-defined): supported features (has media, has comments, has
  revisions), taxonomies attached, capability mapping.

### Media (`schema: media`)
- **Attachment** — `id, storage_key, mime_type, size, uploaded_by,
  alt_text, caption`. The binary lives in object storage; this row is
  metadata.
- **AttachmentRendition** — generated derivatives (thumbnail, medium,
  large, WebP) produced by the media pipeline worker, each with its own
  `storage_key` and dimensions.

### Identity (`schema: identity`)
- **User** — account, credentials (via ASP.NET Core Identity), profile.
- **Role** — named set of **Capabilities** (mirrors WordPress's
  Administrator/Editor/Author/Contributor/Subscriber, plus custom roles).
- **Capability** — a fine-grained permission string (`edit_posts`,
  `edit_others_posts`, `publish_posts`, `manage_options`,
  `manage_plugins`, …), assigned to roles, checked via policy-based
  authorization on every write.
- **UserSession / RefreshToken** — issued by the Identity module (OpenIddict).

### Comments (`schema: comments`)
- **Comment** — `post_id, author (user or guest name/email), content,
  status (pending/approved/spam/trash), parent_id (threading)`.
- **CommentMeta** — same JSONB pattern as PostMeta.
- Moderation queue, Akismet-equivalent spam-check hook point (pluggable —
  see Extensibility below).

### Navigation (`schema: navigation`)
- **Menu**, **MenuItem** (link, post reference, or custom URL; nested for
  multi-level menus).
- **Widget**, **Sidebar** — named widget areas a theme exposes, and the
  widgets placed in them.

### Appearance (`schema: appearance`)
- **Theme** — registered theme package (see
  [05-frontend-architecture.md](05-frontend-architecture.md) for what a
  "theme" is in a React front office).
- **ThemeSetting / CustomizerSetting** — JSONB key/value settings scoped to
  the active theme (colors, logo, layout options).

### Extensibility (`schema: extensibility`)
- **Plugin** — registered plugin package: id, version, status
  (active/inactive), declared hook subscriptions, declared capabilities it
  requires.
- **PluginSetting** — JSONB settings scoped to a plugin.
- **HookRegistration** — which domain events / REST webhooks a plugin
  subscribes to.
- See [ADR list](../adr) and
  [06-auth-and-security.md](06-auth-and-security.md) for how plugin code
  actually executes (sandboxing).

### Settings (`schema: settings`)
- **Option** — global site settings as key/JSONB value (site title,
  tagline, permalink structure, timezone, default post format) — the
  `wp_options` equivalent, without the "anything goes" untyped blob
  problem: each known key has a schema validated at write time.

### Search (`schema: search`)
- **SearchDocument** — denormalized projection of searchable content
  (post title/body/excerpt/taxonomy terms), kept in sync via domain events.
  Backed by Postgres full-text search (`tsvector`) in Phase 1–4; the same
  projection can be shipped to OpenSearch/Elasticsearch later without
  changing the write side (see [03-database-design.md](03-database-design.md)).

### Audit (`schema: audit`)
- **AuditLogEntry** — who did what, when, to which entity — required for
  admin accountability and for the Site-Health-style status page.

## Cross-module interaction rules

1. A module's EF Core `DbContext` may only reference tables in its own
   schema. No cross-schema joins in application code.
2. Need data from another module? Call its public query handler (MediatR
   `IRequestHandler`), not its `DbContext`.
3. Need to react to something another module did? Subscribe to its domain
   event (e.g., Comments subscribes to `PostDeleted` to cascade-trash its
   comments).
4. Only the **Content**, **Media**, **Comments**, and **Navigation** modules
   are exposed to the plugin hook system for MVP (see
   [ADR](../adr) and the roadmap's Extensibility phase) — Identity and
   Settings are read-only to plugins at first, to keep the security surface
   small.

## Entity ownership vs. WordPress equivalents

| WordPress concept | CleanPress equivalent |
|---|---|
| `wp_posts` | `Content.Post` |
| `wp_postmeta` (serialized PHP) | `Content.PostMeta` (typed JSONB) |
| `wp_terms` / `wp_term_taxonomy` | `Content.Taxonomy` / `Content.Term` |
| `wp_users` / `wp_usermeta` | `Identity.User` |
| Roles & Capabilities (PHP arrays) | `Identity.Role` / `Identity.Capability` (data-driven) |
| `wp_comments` | `Comments.Comment` |
| `wp_options` | `Settings.Option` (schema-validated) |
| Menus / Widgets | `Navigation.Menu` / `Navigation.Widget` |
| Themes (PHP templates) | `Appearance.Theme` (React component package) + `ThemeSetting` |
| Plugins (arbitrary PHP) | `Extensibility.Plugin` (sandboxed, hook-scoped) |
| Actions & Filters (hooks) | Domain events + defined extension points (see extensibility phase) |
| WP-Cron | Hangfire recurring jobs |
