# ADR-0007: OpenIddict for self-hosted identity, not a managed cloud IdP

## Status
Accepted

## Context
The platform needs an OAuth2/OIDC identity provider for two SPAs (Authorization
Code + PKCE) and server-to-server auth (webhook/plugin client credentials —
see [06-auth-and-security.md](../architecture/06-auth-and-security.md#flows)).
Managed options (Azure AD B2C, AWS Cognito, Auth0, Okta) would reduce
implementation effort but are each tied to a specific vendor.

## Decision
Use **OpenIddict**, a .NET-native, open-source OAuth2/OIDC server, hosted
as a module inside the backend (see
[02-domain-model.md](../architecture/02-domain-model.md#identity-schema-identity)).

## Consequences

**Positive**
- Consistent with the cloud-agnostic hosting decision
  ([ADR-0001](0001-cloud-agnostic-kubernetes-hosting.md)) — no
  per-environment identity provider configuration divergence, no
  dependency on a specific cloud region/service being available.
- Native .NET integration (ASP.NET Core Identity underneath) — one
  language/runtime for the whole backend, one team skill set, direct
  access to the same `Identity.User`/`Role`/`Capability` data model rather
  than syncing users into an external directory.
- No per-monthly-active-user vendor pricing, which matters for a
  self-hostable, potentially self-operated CMS product.
- Architected (see [ADR-0004](0004-modular-monolith-backend.md)) as the
  module most likely to be split into its own deployable first — the
  self-hosted choice doesn't foreclose that, a managed-IdP choice would
  have made it a rewrite instead.

**Negative / trade-offs**
- The team owns identity-server correctness and security (token signing
  key rotation, session/refresh-token storage, MFA implementation) instead
  of delegating it to a vendor with a dedicated security team. Mitigated
  by OpenIddict being a mature, widely-audited library rather than a
  homegrown implementation, and by the OWASP ASVS baseline in
  [06-auth-and-security.md](../architecture/06-auth-and-security.md#transport-and-infrastructure-security).
- No out-of-the-box social login / enterprise SSO (SAML, external OIDC
  federation) — deferred; OpenIddict supports federating to external IdPs
  as an *upstream* provider if/when that's needed, without changing what
  the frontends talk to.

## Alternatives considered
- **Azure AD B2C / AWS Cognito**: rejected — directly cloud-specific,
  contradicting [ADR-0001](0001-cloud-agnostic-kubernetes-hosting.md).
- **Auth0 / Okta**: cloud-agnostic in principle (SaaS, not tied to a
  specific infra cloud) but still a third-party managed dependency and
  per-MAU cost; rejected primarily on cost/control grounds for a
  self-hostable product, revisitable if a managed IdP's reduced
  implementation effort outweighs that once real usage numbers exist.
- **IdentityServer (Duende)**: a strong alternative, functionally similar
  to OpenIddict; OpenIddict was chosen for its permissive open-source
  license (Duende IdentityServer's commercial licensing for
  production use at scale is a cost/legal dependency this decision
  specifically avoids).
