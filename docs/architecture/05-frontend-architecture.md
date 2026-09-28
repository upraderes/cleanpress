# 05 — Frontend Architecture

Two separate applications, sharing packages but not a runtime — they have
fundamentally different requirements (SEO/performance for the public site
vs. rich interactivity for the dashboard), so forcing one framework
decision onto both would compromise one of them.

## Front office — `frontend-public`

- **Framework**: Next.js (App Router), TypeScript, React Server Components
  where they help (reduce client JS for mostly-static content), client
  components for interactive islands (comment forms, search-as-you-type).
- **Rendering strategy**:
  - Published posts/pages: **ISR** (Incremental Static Regeneration) with
    **on-demand revalidation** — the backend calls Next.js's revalidate
    endpoint (authenticated via a shared secret) when content is
    published/updated (see the publish flow in
    [01-system-architecture.md](01-system-architecture.md#request-flow-examples)),
    so pages are as static/cacheable as WordPress + a page-cache plugin,
    without ever serving stale content after an edit.
  - Search results, logged-in-member views: SSR, no static caching.
- **SEO**: automatic `sitemap.xml` and `robots.txt` generation from the
  Content module, per-page `<title>`/meta description/canonical URL driven
  by post fields, Open Graph + Twitter Card tags, JSON-LD structured data
  (Article/BlogPosting schema.org) — all first-class fields on Post, not a
  bolted-on plugin.
- **Permalinks**: configurable permalink structure
  (`/%postname%/`, `/%year%/%month%/%postname%/`, custom), stored in
  `Settings.Option`, resolved server-side via a catch-all Next.js route
  that asks the API to resolve a path to a Post.
- **Theming (MVP)**: a **theme** is a Next.js "theme package" —
  layout components + a design-token stylesheet (CSS variables) — selected
  at **build/deploy time** via configuration (one theme active per site,
  matching the single-site scope in
  [ADR-0003](../adr/0003-single-site-not-multi-tenant.md)). Dynamic,
  runtime theme switching without a redeploy is a Phase 6+ concern (see
  [roadmap](../roadmap.md)), likely via Module Federation once there's a
  real theme marketplace to serve.
- **Performance budget**: Lighthouse CI in the pipeline enforces a budget
  (see [09-testing-strategy.md](09-testing-strategy.md)); images are always
  served through the media pipeline's responsive renditions (see
  [02-domain-model.md](02-domain-model.md#media-schema-media)) via
  `next/image`.

## Back office — `frontend-admin`

- **Framework**: React + TypeScript, built with Vite. Pure SPA — no SSR
  need; served as static assets behind the same ingress, authenticated
  before any meaningful content loads.
- **Server state**: TanStack Query against the generated
  `packages/api-client`, giving cache invalidation, optimistic updates
  (e.g. instant UI feedback on publish, rolled back on 409), and request
  deduplication for free.
- **UI foundation**: Radix UI primitives + Tailwind CSS for accessible,
  unstyled-by-default components, themed via the same design-token
  approach as the front office (shared `packages/ui-kit` tokens) — the
  admin doesn't need to *look* like the active site theme, but reusing the
  token pipeline avoids a second design system.
- **Block editor**: built on **TipTap/ProseMirror** — a structured,
  extensible document model (the Gutenberg equivalent). Each block type is
  a registered `{ schema, React component, toolbar }` triple; built-in
  blocks (paragraph, heading, image, gallery, embed, list, quote, code)
  ship in Phase 3, with the same registration API available to plugins in
  Phase 4 (see [roadmap](../roadmap.md)).
- **Editing locks**: a short-lived Redis lock + heartbeat (matching
  [03-database-design.md](03-database-design.md)'s optimistic concurrency
  story) drives a "Jane is currently editing this post" banner, avoiding
  silent overwrite the way plain optimistic concurrency alone would still
  allow two people to start editing before either saves.
- **RBAC-aware UI**: capability checks (from
  [02-domain-model.md](02-domain-model.md#identity-schema-identity)) hide
  or disable actions the current user's role can't perform, in addition to
  (never instead of) the backend enforcing the same policies — the backend
  is always the actual authorization boundary.
- **Media library**: grid/list views, drag-drop upload directly to a
  pre-signed object-storage URL (bypassing the API for the byte stream
  itself; the API only records metadata and triggers the rendition
  pipeline) — keeps large uploads off the API's own request pipeline.

## Shared packages (`packages/`)

| Package | Purpose |
|---|---|
| `api-client` | Generated TypeScript client + hooks from the OpenAPI spec. Regenerated in CI whenever the backend's spec changes; a diff here that neither frontend's code adapts to fails CI (see [04-api-design.md](04-api-design.md#versioning-policy)). |
| `ui-kit` | Shared design tokens, primitive components, icons — consumed by `frontend-admin` directly and by `frontend-public` theme packages for token values. |
| `config` | Shared `tsconfig`, ESLint, Prettier config so both apps and all packages lint/typecheck identically. |

## Monorepo tooling

pnpm workspaces + Turborepo for task orchestration and build caching
across `frontend-public`, `frontend-admin`, and `packages/*` — keeps CI
fast by only rebuilding/retesting what actually changed, and keeps
dependency versions (React, TypeScript) consistent across both apps.
