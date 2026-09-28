# ADR-0004: Modular monolith backend, not microservices

## Status
Accepted

## Context
The backend needs to host content, media, identity, comments, navigation,
appearance, extensibility, settings, search, and audit logic. Microservices
would give independent scaling and deployment per concern; a single
deployable is operationally simpler but risks becoming an unstructured
"big ball of mud" if module boundaries aren't enforced.

## Decision
Build the backend as a **modular monolith**: one deployable (see
[01-system-architecture.md](../architecture/01-system-architecture.md)),
internally split into modules that each own their own database schema,
`DbContext`, and migrations, and that only interact through in-process
mediated queries and domain events — never direct cross-module data access.
See [02-domain-model.md](../architecture/02-domain-model.md#cross-module-interaction-rules).
Boundaries are enforced in CI via architecture tests (see
[09-testing-strategy.md](../architecture/09-testing-strategy.md#backend-backend)),
not just convention.

## Consequences

**Positive**
- One deployable to build, test, deploy, and operate for MVP — no service
  mesh, no distributed transaction/saga complexity, no network calls
  between modules that are really just business logic.
- Strict module boundaries mean any module *can* be extracted into its own
  service later (Identity is the most likely first candidate — see
  [06-auth-and-security.md](../architecture/06-auth-and-security.md#identity-provider))
  by moving its schema to its own database and swapping its in-process
  mediator calls for network calls, without touching its internal logic.
- Simpler local development: one process (or two, with the worker) to run
  instead of ten.
- Simpler transactional integrity for cross-module-adjacent operations
  that would otherwise need distributed transactions (e.g. publish
  touching Content, and Search's projection update reacting via an
  in-process event with an outbox pattern instead of a distributed saga).

**Negative / trade-offs**
- All modules currently scale together (same pod, same resource limits) —
  a module with very different load characteristics (e.g. media
  processing) can't be scaled independently without extraction. Mitigated
  today by running the media pipeline as a **separate worker deployable**
  from the start (see
  [01-system-architecture.md](../architecture/01-system-architecture.md#components)),
  since it was the one module with an obviously different resource profile
  (CPU-bound image/video processing vs. the API's I/O-bound request
  handling).
- Requires discipline (and the architecture-test enforcement above) to
  avoid boundaries eroding over time — the main risk of the modular
  monolith approach in practice.
- A single bug/crash can, in the worst case, affect the whole API process,
  unlike an isolated microservice failure. Mitigated by standard resilience
  practices (multiple replicas, readiness probes pulling unhealthy pods
  from rotation — see
  [07-infrastructure-and-deployment.md](../architecture/07-infrastructure-and-deployment.md#rollout-strategy)).

## Alternatives considered
- **Microservices from day one**: rejected for MVP — the operational
  overhead (service discovery, distributed tracing being load-bearing
  rather than nice-to-have, per-service CI/CD, data consistency across
  service boundaries) isn't justified before there's a single working core
  CMS feature, and the team/scale context given doesn't call for it.
- **Unstructured monolith** (no enforced module boundaries): rejected —
  this is the failure mode modular-monolith discipline exists to avoid;
  without it, a later extraction becomes a rewrite instead of a
  refactor.
