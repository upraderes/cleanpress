# Roadmap

Phased, not calendar-dated — no team size or timeline was specified. Each
phase lists an entry condition, scope, and exit criteria; once a team size
and delivery cadence are known, phases can be mapped onto sprints/quarters.
See [feature-matrix.md](feature-matrix.md) for the WordPress-parity view of
the same plan.

## Phase 0 — Foundations

**Entry**: this plan is approved.
**Scope**:
- Monorepo scaffolding: `backend/` (.NET solution, empty modules with
  boundary tests wired up), `frontend-public/` (Next.js skeleton),
  `frontend-admin/` (Vite skeleton), `packages/` (api-client, ui-kit,
  config), pnpm workspaces + Turborepo.
- CI: lint/typecheck/unit-test pipelines for all four projects; CodeQL and
  dependency scanning enabled from commit one.
- Local dev: docker-compose stack (Postgres, Redis, MinIO) + `kind`/Tilt
  setup.
- Infra skeleton: base Helm charts, Terraform module structure (even if
  only the `dev`/local provider is implemented first), CloudNativePG and
  Redis operator installed in a dev cluster.
- Identity module: OpenIddict wired up, one seeded Administrator account,
  Authorization Code + PKCE working end to end from `frontend-admin`
  against an empty API.
**Exit criteria**: an authenticated admin can log into an empty back
office; CI is green on an empty-but-structured monorepo; `dev` environment
deploys via GitOps.

## Phase 1 — Core content management

**Scope**: Content module — posts, pages, categories, tags, custom
taxonomies (data model only, UI for defining new ones is Phase 3+), media
library (upload, storage, basic renditions), draft/publish/schedule/trash
lifecycle, revisions. Back office: minimal list/edit UI (rich block editor
lands in Phase 3 — Phase 1's editor can be a single rich-text field to
unblock end-to-end content flow sooner). Front office: server-rendered post
and page views, no theme system yet (one hard-coded layout).
**Exit criteria**: an admin can create, publish, and edit a post with
media, and it renders correctly (and is revalidated on edit) on the public
site.

## Phase 2 — Users, roles, comments, SEO basics

**Scope**: Identity module — roles/capabilities matching WordPress
defaults, user management UI, capability-gated API/UI throughout. Comments
module — public comment submission, moderation queue, threading. Front
office SEO: sitemap.xml, robots.txt, meta tags, Open Graph, JSON-LD,
configurable permalink structure. Account data export/erasure endpoints
(see [06-auth-and-security.md](architecture/06-auth-and-security.md#data-protection)).
**Exit criteria**: multiple users with distinct roles can collaborate on
content with correct capability enforcement; a visitor can comment and an
editor can moderate; the public site is indexable and correctly tagged.

## Phase 3 — Back office UX and theming

**Scope**: full block editor (TipTap/ProseMirror-based, built-in block
set), menus and widgets/sidebars, appearance customizer (theme settings,
build-time theme selection), site settings UI, editing locks
(concurrent-edit protection), Site-Health-style status page.
**Exit criteria**: the back office is usable day-to-day without a
developer in the loop; the public site's look is configurable through the
customizer within the active theme's supported options.

## Phase 4 — Extensibility

**Scope**: Extensibility module — plugin registration, the webhook
delivery pipeline (signed payloads, retry, dead-letter view — see
[04-api-design.md](architecture/04-api-design.md#extensibility-surface-webhooks--domain-events)),
the WASM sandbox runtime for inline content filters (see
[ADR-0006](adr/0006-plugin-sandboxing-not-arbitrary-code.md)), a plugin
settings UI in the back office, block-type registration API opened to
plugins, theme package format documented for third-party themes.
**Exit criteria**: a first-party "reference plugin" (e.g. a simple SEO
enhancer or a contact-form block) is built entirely through the public
extensibility API, proving it's real and not just documented.

## Phase 5 — Search, performance, and hardening

**Scope**: evaluate Postgres FTS against real content volume/query
patterns; migrate to OpenSearch/Elasticsearch if warranted (see
[03-database-design.md](architecture/03-database-design.md#full-text-search)).
Redis fragment/output caching for the public site, CDN integration
guidance, image optimization tuning. Load testing (k6) against realistic
traffic shapes. Backup-restore drills. Argo Rollouts canary deploys.
Synthetic monitoring. Pre-launch penetration test.
**Exit criteria**: the platform meets the SLOs in
[08-observability.md](architecture/08-observability.md#slos-initial-targets-revisited-once-theres-real-traffic)
under realistic load, and a full backup restore has been successfully
rehearsed.

## Phase 6 — Ecosystem and growth

**Scope** (evaluate and prioritize based on actual user demand at this
point, not committed wholesale):
- Plugin/theme marketplace (discovery, versioning, update notifications —
  not necessarily third-party billing).
- E-commerce module (WooCommerce-equivalent), built as a consumer of the
  Phase 4 extensibility API, not a special-cased core feature.
- WordPress import tooling (WXR import) for migrating existing WordPress
  sites onto CleanPress.
- Multisite / multi-tenant SaaS evaluation — revisit
  [ADR-0003](adr/0003-single-site-not-multi-tenant.md) with real demand
  data before committing to the architectural changes it would require.
- GraphQL API (HotChocolate) for headless/flexible-query consumers.
- Dynamic (runtime) theme switching without redeploy.

## Cross-phase, always-on tracks

These aren't a single phase — they run continuously from Phase 0:
- **Security**: dependency scanning, SAST, and the OWASP ASVS baseline
  apply from the first commit, not retrofitted before launch.
- **Accessibility**: WCAG 2.1 AA target for both frontends, checked in the
  same Playwright/CI pipeline as functional e2e tests, from Phase 1.
- **Internationalization**: content model supports future localization
  (WordPress's multilingual-plugin pattern) — the data model reserves for
  it (a `locale` field on Post from Phase 1) even though a full
  translation UI is a Phase 6+ candidate.
- **Documentation**: this `docs/` tree is a living artifact — every ADR and
  architecture doc is updated in the same PR that changes the decision it
  describes, not after the fact.
