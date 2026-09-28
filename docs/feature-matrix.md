# Feature Matrix — WordPress Parity Tracking

Tracks CleanPress's plan against WordPress core feature areas, mapped to
the phase in [roadmap.md](roadmap.md) that delivers each one. This is a
planning artifact — update `Status` as work lands.

Status legend: `Planned` (designed, not built) · `In Progress` · `Done` ·
`Deferred` (explicitly out of scope for now, see rationale) · `Descoped`
(not planned — rationale given).

| Feature area | WordPress core feature | CleanPress plan | Phase | Status |
|---|---|---|---|---|
| **Content** | Posts | `Content.Post`, `post_type = post` | 1 | Planned |
| | Pages (hierarchical) | `Content.Post`, `post_type = page`, `parent_id` | 1 | Planned |
| | Custom post types | `CustomPostTypeDefinition` | 1 (data model), 4 (plugin-registrable) | Planned |
| | Categories & tags | `Taxonomy`/`Term`, built-in taxonomies | 1 | Planned |
| | Custom taxonomies | `Taxonomy`/`Term`, admin-defined | 3 | Planned |
| | Post revisions | `Revision` | 1 | Planned |
| | Autosave / draft recovery | Editor autosave → `Revision` | 3 (with full block editor) | Planned |
| | Trash / restore | `Post.status = trash`, retention job | 1 | Planned |
| | Scheduled publishing | `Post.status = scheduled` + Hangfire job | 1 | Planned |
| | Post formats | `PostMeta`-driven, theme-rendered | 3 | Planned |
| | Sticky posts | `PostMeta` flag + query support | 3 | Planned |
| **Editor** | Block editor (Gutenberg) | TipTap/ProseMirror-based block editor | 3 | Planned |
| | Custom HTML block | Sanitized on save (see [06-auth-and-security.md](architecture/06-auth-and-security.md#content-security)) | 3 | Planned |
| | Reusable blocks / patterns | Saved block templates | 4+ | Deferred — post-MVP UX nicety |
| **Media** | Media library | `Attachment` + object storage | 1 | Planned |
| | Image thumbnails/renditions | `AttachmentRendition`, worker pipeline | 1 | Planned |
| | Featured image | `PostMeta` reference to `Attachment` | 1 | Planned |
| | Galleries | Gallery block type | 3 | Planned |
| **Users** | Users & profiles | `Identity.User` | 2 | Planned |
| | Roles & capabilities | `Role`/`Capability`, WP-default roles + custom | 2 | Planned |
| | Multi-author collaboration | Capability-gated content ownership | 2 | Planned |
| | User registration (public) | Front-office self-registration, `Subscriber` role | 2 | Planned |
| | MFA | TOTP, required for Administrator | 2/3 | Planned |
| **Comments** | Comments | `Comments.Comment`, threading | 2 | Planned |
| | Moderation queue | Status workflow + back-office UI | 2 | Planned |
| | Spam filtering (Akismet-equiv.) | Pluggable spam-check hook point | 2 (hook point), 4 (real plugin) | Planned |
| **Navigation** | Menus | `Navigation.Menu`/`MenuItem` | 3 | Planned |
| | Widgets / sidebars | `Navigation.Widget`/`Sidebar` | 3 | Planned |
| **Appearance** | Themes | React theme packages (`Appearance.Theme`) | 1 (single hard-coded), 3 (customizer) | Planned |
| | Customizer | `ThemeSetting`/`CustomizerSetting` UI | 3 | Planned |
| | Dynamic theme switching (no redeploy) | Module Federation-based runtime loading | 6 | Deferred — see [ADR-0003](adr/0003-single-site-not-multi-tenant.md) forward-compat notes |
| | Widgetized footer/sidebar areas | Theme-declared widget areas | 3 | Planned |
| **Extensibility** | Plugins (arbitrary PHP) | Webhooks + WASM sandbox (see [ADR-0006](adr/0006-plugin-sandboxing-not-arbitrary-code.md)) | 4 | Planned (deliberately narrower than WP — see ADR) |
| | Actions/filters (hooks) | Domain events (first-party) + webhooks/WASM (plugins) | 4 | Planned |
| | Plugin/theme marketplace | Discovery + versioning UI | 6 | Deferred |
| **Settings** | Site settings (`wp_options`) | `Settings.Option`, schema-validated | 1 | Planned |
| | Permalink structure | Configurable, resolved server-side | 2 | Planned |
| | Site Health | `/api/v1/site-health` + back-office status page | 3 | Planned |
| **Search** | Search | Postgres FTS → OpenSearch if warranted | 1 (basic), 5 (scaled) | Planned |
| **Performance/Caching** | Page caching | Next.js ISR + Redis fragment cache | 1 (ISR), 5 (Redis tuning) | Planned |
| | CDN support | Provider-agnostic, any CDN in front of ingress | 5 | Planned |
| **REST API** | WP REST API | `/api/v1/*`, OpenAPI-generated (see [04-api-design.md](architecture/04-api-design.md)) | 1+ (grows with each module) | Planned |
| | GraphQL (WPGraphQL-equivalent) | HotChocolate | 6 | Deferred |
| **Import/Export** | Import/export (WXR) | WordPress WXR import tooling | 6 | Deferred |
| **Multisite** | Network of sites | Multi-tenant SaaS re-architecture | 6 (evaluation only) | Deferred — see [ADR-0003](adr/0003-single-site-not-multi-tenant.md) |
| **Cron** | WP-Cron | Hangfire recurring jobs | 1 | Planned |
| **E-commerce** | WooCommerce | Dedicated module, built on Phase 4 extensibility API | 6 | Deferred |
| **Forms** | Contact forms (plugin territory in WP too) | Reference plugin example | 4 | Planned (as extensibility proof, see [roadmap](roadmap.md)) |
| **Accessibility** | Varies by theme/plugin | WCAG 2.1 AA target, both frontends, CI-enforced | Continuous from 1 | Planned |
| **i18n/l10n** | WPML/Polylang (plugin territory) | `locale` field reserved from Phase 1; full translation UI later | 1 (reserved), 6 (UI) | Deferred |
| **Security** | Varies widely by install hygiene | OWASP ASVS L2 baseline, sandboxed plugins by design | Continuous from 0 | Planned |
| **Backups** | Plugin territory in WP | WAL archiving + Velero, tested restores | 0 (mechanism), 5 (drilled) | Planned |
| **Data privacy** | Personal data export/erasure | Identity module orchestrated erasure | 2 | Planned |

## Descoped (not planned, with rationale)

| WordPress feature | Why descoped |
|---|---|
| Arbitrary PHP-equivalent plugin code execution | Security trade-off — see [ADR-0006](adr/0006-plugin-sandboxing-not-arbitrary-code.md). Webhooks + WASM cover the majority of real extensibility needs with a much smaller risk surface. |
| Classic (non-block) editor / TinyMCE | No reason to build two editor paradigms; the block editor is the only editor. |
| Theme/plugin marketplace billing | Out of scope until there's a marketplace at all (Phase 6) and real demand to monetize it. |
