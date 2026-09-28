# ADR-0005: Next.js for the front office, Vite SPA for the back office

## Status
Accepted

## Context
Both frontends are React + TypeScript (stakeholder requirement), but they
have opposite requirements: the front office is public, SEO-critical, and
needs fast first paint for anonymous visitors; the back office is
authenticated-only, has no SEO requirement, and prioritizes rich
interactivity (a block editor, media library, live data).

## Decision
Use two separate applications rather than one framework/codebase for both:
- **`frontend-public`**: Next.js (App Router), server-rendered with ISR +
  on-demand revalidation. See
  [05-frontend-architecture.md](../architecture/05-frontend-architecture.md#front-office--frontend-public).
- **`frontend-admin`**: React + Vite SPA, no SSR. See
  [05-frontend-architecture.md](../architecture/05-frontend-architecture.md#back-office--frontend-admin).

Both consume the same generated API client (`packages/api-client`) and
share a design-token/component package (`packages/ui-kit`), so the split
doesn't mean duplicated frontend logic — only a different rendering shell.

## Consequences

**Positive**
- Front office gets WordPress-competitive SEO and performance
  (server-rendered HTML, cacheable via ISR, Core Web Vitals monitored —
  see [08-observability.md](../architecture/08-observability.md#what-gets-measured))
  without forcing the admin dashboard through the same (unnecessary)
  server-rendering complexity.
- Back office gets a simpler deployment (static assets) and a simpler
  mental model (pure client-rendered SPA, no hydration mismatches to
  debug) since it never needs to be indexed or fast-first-paint for an
  anonymous visitor.
- Each app can adopt framework upgrades independently — a Next.js major
  version bump doesn't force touching the admin SPA and vice versa.

**Negative / trade-offs**
- Two build pipelines, two sets of framework-specific conventions/config
  for contributors to learn, instead of one.
- Shared UI components must be framework-neutral (no Next.js-specific APIs
  leaking into `packages/ui-kit`) to actually be shared — an ongoing
  discipline requirement, checked via `packages/config`'s shared lint
  rules.

## Alternatives considered
- **One Next.js app for both, admin under `/admin`**: rejected — the
  admin routes would still need to opt out of SSR/ISR for nearly every
  page (auth-gated, highly dynamic), gaining little from sharing a
  framework while inheriting Next.js's routing/caching model for pages
  that don't want it.
- **One Vite SPA for both, front office pre-rendered separately**: rejected
  — building competitive SEO (dynamic sitemaps, on-demand revalidation on
  publish, streaming SSR) without a framework that provides it natively
  would mean re-implementing a large slice of what Next.js already solves.
