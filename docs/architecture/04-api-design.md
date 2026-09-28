# 04 — API Design

## Style: REST, code-first, OpenAPI as the generated contract

The API is REST over JSON, versioned in the URL path (`/api/v1/...`).
Controllers/endpoints are written in C# (minimal APIs grouped by module);
Swashbuckle generates the OpenAPI spec from them. Both frontends consume a
**generated TypeScript client** (`openapi-typescript` + a thin fetch
wrapper) published as `packages/api-client` — no hand-written duplicate
DTOs on the frontend, and a breaking API change is a visible TypeScript
compile error in both frontend apps' CI.

GraphQL (HotChocolate) is deliberately **not** in the MVP — see the
roadmap's later phases. It's the right tool once the public front office
needs flexible, client-shaped queries (a WPGraphQL equivalent for headless
consumers/plugins), but it's a second API surface to design and secure, and
REST covers every MVP need.

## Conventions

- **Resource-oriented URLs**: `/api/v1/posts`, `/api/v1/posts/{id}`,
  `/api/v1/posts/{id}/revisions`, `/api/v1/media`, `/api/v1/users`, etc.
  Actions that aren't pure CRUD are modeled as sub-resources or POST verbs
  on the resource (`POST /api/v1/posts/{id}/publish`,
  `POST /api/v1/posts/{id}/trash`), not RPC-style endpoints.
- **Pagination**: cursor-based (`?cursor=...&limit=...`) for anything that
  can grow unbounded (post lists, comments, audit log). Offset pagination
  is not used — it doesn't hold up under concurrent writes to the same list.
- **Filtering/sorting**: a small, explicit query-parameter vocabulary per
  resource (`?status=published&orderBy=publishedAt:desc`), not a generic
  query language, to keep authorization checks and index usage predictable.
- **Errors**: [RFC 7807 Problem Details](https://www.rfc-editor.org/rfc/rfc7807)
  for every non-2xx response — machine-readable `type`, human `title`,
  `status`, and a `errors` map for field-level validation failures
  (FluentValidation output mapped directly into this shape).
- **Idempotency**: mutating endpoints accept an optional
  `Idempotency-Key` header, honored for at least 24h, so retried
  publishes/uploads from a flaky client connection don't double-create.
- **ETags / concurrency**: read endpoints for mutable resources return an
  `ETag`; writes require `If-Match`, mapped to the EF Core `xmin`
  concurrency token (see
  [03-database-design.md](03-database-design.md#naming-and-types)) — a 409
  Conflict surfaces the "someone else edited this" case to the UI.
- **Rate limiting**: ASP.NET Core's built-in rate-limiting middleware,
  policies per endpoint class (auth endpoints stricter than reads),
  backed by Redis so limits are consistent across API replicas.

## Public vs. authenticated reads

Rather than a separate BFF (see
[01-system-architecture.md](01-system-architecture.md#why-one-api-instead-of-an-api-per-frontend-bff)),
the same resource exposes different shapes/authorization by context:

- `GET /api/v1/posts?status=published` — anonymous-accessible, served from
  the Postgres read replica, cacheable.
- `GET /api/v1/posts?status=draft` — requires `edit_posts` capability,
  always reads primary, never cached.
- The API layer enforces this via policy-based authorization
  (`[Authorize(Policy = "CanViewDraftContent")]`), not by trusting a
  frontend to only ask for what it's allowed to see.

## Extensibility surface: webhooks + domain events

WordPress's "actions and filters" become, in CleanPress:

- **Domain events** (in-process, via MediatR) for first-party modules
  reacting to each other — e.g. Comments reacting to `PostDeleted`.
- **Outbound webhooks** for plugins: a plugin registers interest in named
  events (`post.published`, `comment.created`, `user.registered`, …) via
  `Extensibility.HookRegistration`; the backend POSTs a signed payload
  (HMAC over the body, like Stripe/GitHub webhooks) to the plugin's
  registered endpoint, delivered by the background worker with retry +
  backoff and a dead-letter view in the back office.
- **Filter-equivalent (content transformation) hooks** are a Phase 4+
  problem — see [roadmap](../roadmap.md) and
  [06-auth-and-security.md](06-auth-and-security.md) for why these need a
  sandboxed execution model (WASM) rather than plain HTTP webhooks: a
  filter must run synchronously, in-line with the request, which a
  network round-trip to arbitrary third-party code cannot safely guarantee.

## Versioning policy

- `v1` is additive-only for as long as possible: new optional fields, new
  endpoints. A breaking change (removed/renamed field, changed semantics)
  requires a new version segment (`/api/v2/...`), with `v1` kept running
  until both frontends and any registered plugins have migrated.
- The OpenAPI spec is diffed in CI against the previous release; an
  unannounced breaking change fails the build.

## Health and meta endpoints

- `GET /healthz` — liveness (process is up).
- `GET /readyz` — readiness (DB, Redis, object storage reachable) — wired
  to Kubernetes probes, see
  [07-infrastructure-and-deployment.md](07-infrastructure-and-deployment.md).
- `GET /api/v1/site-health` — authenticated, back-office-facing: the
  WordPress "Site Health" equivalent (PHP-version-style checks, but for
  this stack: migration status, background job queue depth, webhook
  delivery failures, disk/storage usage).
