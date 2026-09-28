# ADR-0006: Plugin extensibility via webhooks + WASM sandbox, not arbitrary server code

## Status
Accepted (scope: applies once the Extensibility phase begins — see
[roadmap](../roadmap.md); recorded now because it constrains the domain
model and API design from Phase 0).

## Context
WordPress's plugin ecosystem is its single biggest strength and its
single biggest security liability: plugins are arbitrary PHP running
in-process with full access to the database and filesystem, and the
long-running result is that the large majority of real-world WordPress
compromises trace back to vulnerable or malicious plugins/themes, not
WordPress core itself. CleanPress needs an extensibility story that
captures most of the *value* (third parties can meaningfully extend
behavior) without inheriting that risk profile wholesale.

## Decision
Two supported extension mechanisms, no third:
1. **Outbound webhooks** for anything that can be async/out-of-process:
   the backend calls a plugin's own server on named domain events
   (`post.published`, `comment.created`, …), signed with HMAC, delivered
   with retry by the background worker. See
   [04-api-design.md](../architecture/04-api-design.md#extensibility-surface-webhooks--domain-events).
2. **WASM-sandboxed inline extensions** (Wasmtime) for logic that must run
   synchronously in-request (content filters/transforms) — a narrow,
   capability-scoped host API, no ambient filesystem/network/database
   access. See
   [06-auth-and-security.md](../architecture/06-auth-and-security.md#plugin-sandboxing).

Plugins never get a loaded .NET assembly running with the API's own
privileges, and never get raw SQL/ORM access to any module's schema.

## Consequences

**Positive**
- Removes an entire class of RCE/supply-chain risk (a compromised or
  malicious plugin cannot read another site's data, write arbitrary files,
  or pivot into the host's network) — a materially better security
  starting point than the ecosystem CleanPress is modeled on.
- Webhooks are a well-understood, language-agnostic integration model —
  plugin authors aren't required to write WASM-target code for the common
  case (react to an event, call back out).
- WASM gives a real answer for the synchronous-filter use case
  (WordPress's `the_content` filter, an SEO plugin rewriting meta tags)
  without opening the sandbox door all the way.

**Negative / trade-offs**
- Strictly less capable than WordPress's plugin model for the cases that
  genuinely need deep, arbitrary integration (e.g. a plugin that wants to
  add its own database tables and admin UI wholesale). Those become
  full first-party modules or forks, not installable plugins, under this
  model.
- WASM sandboxing is more implementation effort than "just load the
  assembly," and the host API surface (what capabilities a sandboxed
  filter can call) has to be deliberately designed and will always be
  narrower than "everything."
- This is explicitly a product trade-off, not a technical inevitability —
  flagged here so it's revisited deliberately (with real plugin-author
  feedback once Phase 4 is underway — see [roadmap](../roadmap.md)) rather
  than silently assumed correct forever.

## Alternatives considered
- **Arbitrary .NET plugin assemblies loaded in-process** (closest
  analogue to WordPress's model): rejected — reintroduces the exact risk
  profile this decision exists to avoid, for a single-site CMS where the
  operator is trusting third-party plugin code with full access to their
  entire site's data.
- **No extensibility model at all (fully closed platform)**: rejected —
  contradicts the stated goal of extensibility as a first-class concern
  (see [00-overview.md](../architecture/00-overview.md#goals)); a CMS
  publishers can't extend isn't a credible WordPress alternative.
