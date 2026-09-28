# 06 — Auth and Security

## Identity provider

**OpenIddict**, hosted as a module inside the backend (see
[02-domain-model.md](02-domain-model.md#identity-schema-identity)),
deliberately chosen over a managed cloud identity service (Auth0, Cognito,
Azure AD B2C) to stay consistent with the cloud-agnostic hosting decision
(see [ADR-0001](../adr/0001-cloud-agnostic-kubernetes-hosting.md)) — an
open-source, self-hosted OIDC/OAuth2 server that runs anywhere the rest of
the platform does. It is architected as the one module that *could* be
split into its own deployable early (auth is a natural, low-fan-in seam),
without being split for MVP.

## Flows

| Client | Flow | Notes |
|---|---|---|
| `frontend-admin` (SPA) | Authorization Code + PKCE | Tokens held in memory, refresh via silent iframe/rotation, never in `localStorage`. |
| `frontend-public` (member login, comments) | Authorization Code + PKCE | Same pattern; anonymous browsing needs no token at all. |
| Backend-to-backend (webhook delivery verification, plugin server calls) | Client Credentials | Scoped, short-lived tokens per registered plugin. |
| `frontend-public` SSR → API (public reads) | None — anonymous | No token issued for anonymous, cacheable, published-content reads. |

## Password and account security

- ASP.NET Core Identity for credential storage, hashing via **Argon2id**
  (configured over the PBKDF2 default).
- Account lockout after repeated failed attempts; login endpoints under a
  stricter rate-limit policy (see
  [04-api-design.md](04-api-design.md#conventions)).
- TOTP-based MFA, required for the Administrator role, optional for others
  — sequenced in the roadmap alongside the rest of the back office
  (see [roadmap](../roadmap.md)).

## Authorization model: roles and capabilities

Mirrors WordPress's role/capability split exactly (see
[02-domain-model.md](02-domain-model.md#identity-schema-identity)):
default roles (Administrator, Editor, Author, Contributor, Subscriber) are
just data — a named bundle of capability strings — and custom roles are a
first-class, data-driven feature, not a code change. Every mutating API
endpoint is decorated with a capability-based authorization policy
(`edit_posts`, `edit_others_posts`, `publish_posts`, `manage_options`,
`manage_plugins`, `moderate_comments`, …); the frontend hides/disables UI
for capabilities the user lacks, but that's a UX nicety, never the actual
boundary — see
[05-frontend-architecture.md](05-frontend-architecture.md#back-office--frontend-admin).

## Content security

- **Block editor output** is structured JSON, not raw HTML from a
  `contenteditable` — the renderer maps known block types to known
  components, which by construction can't emit `<script>` or event-handler
  attributes. Any block type that *does* accept raw HTML (a "Custom HTML"
  block, matching WordPress) is sanitized server-side on save with an
  allowlist sanitizer (tags/attributes), not just client-side.
- **File uploads**: content-type sniffing (not trusting the client's
  declared MIME type), file-size limits, and an extensible malware-scan
  hook point in the media pipeline worker (ClamAV or a cloud scanning API,
  pluggable — not required for MVP, but the interface exists from Phase 1
  so it's a config change later, not a redesign).
- **Output encoding**: front office rendering always encodes
  user-influenced values by default (React does this natively); the only
  path to raw HTML is the sanitized Custom HTML block above.

## Plugin sandboxing

This is the one place CleanPress deliberately does **less** than
WordPress's "arbitrary PHP plugin" model, by design:

- Plugins do not get arbitrary server code execution inside the backend
  process. Extensibility is either **outbound webhooks** (plugin's own
  server, called by ours — see
  [04-api-design.md](04-api-design.md#extensibility-surface-webhooks--domain-events))
  or, for logic that must run inline (content filters/transforms), a
  **WASM sandbox** (Wasmtime) with a narrow, capability-scoped host API —
  no filesystem, no network, no access to other modules' data beyond what
  the hook explicitly passes in.
- This trades some of WordPress's "install anything" flexibility for
  removing an entire class of supply-chain and RCE risk that plagues the
  WordPress plugin ecosystem in practice. It's a conscious product
  trade-off, revisited if real plugin authors hit a wall the sandbox can't
  support — not assumed to be right forever. Sequenced in the roadmap's
  Extensibility phase, not MVP.

## Transport and infrastructure security

- TLS everywhere via cert-manager-issued certificates at the ingress (see
  [07-infrastructure-and-deployment.md](07-infrastructure-and-deployment.md)).
- CORS locked to the known front office/back office origins — no wildcard.
- Secrets (DB credentials, OpenIddict signing keys, object storage keys)
  live in Kubernetes Secrets, sourced from an External Secrets Operator
  integration so the actual secret store (Vault, or any cloud KMS/secret
  manager) stays swappable — never committed, never baked into images.
- Dependency scanning (Dependabot/Renovate + `dotnet list package
  --vulnerable`, `npm audit`) and SAST (CodeQL) run in CI on every PR —
  see [09-testing-strategy.md](09-testing-strategy.md).
- Security baseline target: **OWASP ASVS Level 2** for the auth, content,
  and media modules.

## Auditability

Every admin-capability-gated write is recorded in
`Audit.AuditLogEntry` (actor, action, entity, before/after where
practical, timestamp, IP) — visible in the back office, and the underlying
data for the Site-Health-style status page (see
[04-api-design.md](04-api-design.md#health-and-meta-endpoints)).

## Data protection

Account data export and erasure endpoints (a WordPress core feature since
GDPR) are part of the Identity module from Phase 2 — not deferred to a
later "compliance" phase, since retrofitting erasure across modules that
weren't designed for it is expensive; each module that stores
user-attributable data implements an `IErasesUserData` handler invoked by
a single orchestrated erasure request.
