---
name: cleanpress-react-conventions
description: Use this skill when writing or reviewing any code under frontend-public/, frontend-admin/, or packages/ in the CleanPress repository — a component, a page/route, a data-fetching hook, or a shared package change. Covers the Next.js (front office) vs. Vite SPA (back office) split and shared package conventions specific to this codebase. Load cleanpress-architecture first if the change also involves a design decision, not just implementation within an already-decided pattern.
---

# CleanPress frontend conventions

These implement the decisions in `docs/architecture/05-frontend-architecture.md`
and `docs/adr/0005-nextjs-public-vite-admin-split.md` — read this skill as
"how to write the code," those docs as "why it's split this way."

## Which app does a change belong in?

- **`frontend-public`** (Next.js): anything an anonymous visitor sees —
  post/page rendering, search, comment submission, member login. SEO
  output (meta tags, sitemap, structured data) lives here.
- **`frontend-admin`** (Vite SPA): anything behind authentication for
  managing the site — content editing, media library, users, settings,
  appearance, plugins. Never add SSR/SEO concerns here — it's not needed
  and adds complexity for no benefit.
- **`packages/ui-kit`**: components/tokens used by *both* apps. Must stay
  framework-neutral — no `next/*` imports, no Vite-specific APIs. If a
  component only makes sense in one app, it doesn't belong here.
- **`packages/api-client`**: generated from the backend's OpenAPI spec.
  Never hand-edit generated files — if a type is wrong, the backend
  endpoint/DTO is wrong; fix it there and regenerate.

## `frontend-public` conventions

- Server Components by default; add `"use client"` only for genuinely
  interactive pieces (comment form, search-as-you-type, theme-toggle-style
  widgets) — don't reach for client components out of habit.
- Published content pages use ISR; call the backend's revalidation
  contract (see `docs/architecture/05-frontend-architecture.md#front-office--frontend-public`)
  rather than inventing a different cache-busting mechanism.
- Every page must set title/meta description/canonical/OG tags from
  content fields — never ship a post/page template without them.
- Images always go through `next/image` against the media pipeline's
  renditions (`AttachmentRendition`), never a raw unoptimized `<img>` for
  content images.
- Permalinks are resolved server-side against the configured permalink
  structure — don't hardcode a URL shape into a component.

## `frontend-admin` conventions

- TanStack Query for all server state — no ad-hoc `useEffect` + `fetch`
  data loading. Mutations use TanStack Query's mutation + cache
  invalidation, with optimistic updates only where a 409 (concurrency
  conflict) rollback path is also handled.
- Every action gated by a capability must check that capability
  client-side to hide/disable the control — but this is a UX nicety only;
  never assume the UI check is the security boundary, the backend
  enforces it independently.
- Radix UI primitives + Tailwind, themed via the same design tokens as
  `ui-kit` — don't introduce a second component library or a second token
  source.
- Block editor extensions (new block types) register through the block
  registration API — don't special-case a new block type outside that
  registration mechanism, even for a "core" block, since plugins will use
  the same path in Phase 4.

## Shared conventions

- TypeScript strict mode everywhere; no `any` without a comment explaining
  why it's unavoidable (usually: a genuinely dynamic plugin-provided
  shape).
- Shared lint/tsconfig/prettier config comes from `packages/config` — an
  app-local override is a signal something's wrong with the shared config,
  not a normal thing to add.
- Accessibility: WCAG 2.1 AA is a requirement, not an aspiration (see
  `docs/roadmap.md#cross-phase-always-on-tracks`) — semantic HTML,
  keyboard navigation, and ARIA attributes on custom interactive
  components are expected in the same PR that adds them, not a follow-up.
- New Playwright critical-flow coverage is expected for any new
  user-facing flow that would be embarrassing to silently break (publish
  flow, media upload, login, comment moderation) — see
  `docs/architecture/09-testing-strategy.md`.

## When in doubt

If a component or data-fetching pattern doesn't fit cleanly into the
public/admin split, or seems to need something like runtime theme
switching or a GraphQL query that the current plan defers to a later
phase, check `docs/roadmap.md` and `docs/feature-matrix.md` before
building it — and flag it to the user rather than quietly expanding scope.
