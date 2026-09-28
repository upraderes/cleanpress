---
name: cleanpress-infra
description: Use this skill when writing or reviewing anything under infra/ in the CleanPress repository, or working on CI/CD pipelines (.github/workflows), Dockerfiles, Helm charts, Terraform modules, or Kubernetes manifests for this project. Covers the cloud-agnostic hosting rules and deployment pipeline specific to this codebase. Load cleanpress-architecture first if the change also involves a design decision, not just implementation within an already-decided pattern.
---

# CleanPress infrastructure conventions

These implement `docs/architecture/07-infrastructure-and-deployment.md` and
`docs/adr/0001-cloud-agnostic-kubernetes-hosting.md` — read this skill as
"how to write the infra code," that doc as "why it's structured this way."

## The one rule everything else follows

**Nothing in `infra/helm/` or application config may assume a specific
cloud provider.** Stateful dependencies (Postgres, Redis, object storage)
run via Kubernetes operators (CloudNativePG, a Redis operator, MinIO) so
the same charts deploy identically anywhere. Provider-specific pieces
(DNS, a managed database as an *opt-in* alternative, cloud object storage
credentials) live only in `infra/terraform/providers/{provider}/` — never
leak a provider-specific API call, SDK, or environment assumption into
`backend/`, `frontend-public/`, or `frontend-admin/` application code.
If you find yourself importing an AWS/Azure/GCP SDK into application code,
stop and check whether it belongs behind an existing interface
(`IMediaStorage` for storage, for example) instead.

## Directory structure

```
infra/
  helm/
    api/                 the .NET API deployment
    worker/               background job processor
    migrator/              migration Job, gates api rollout
    frontend-public/       Next.js server deployment
    frontend-admin/        static asset serving
    environments/           umbrella charts per env (dev/staging/prod), pinning versions
  terraform/
    providers/
      aws/ azure/ gcp/ on-prem/    thin, swappable: cluster + DNS + cert issuer + (optional) managed DB/storage
```

## Deploy pipeline rules

- CI builds and pushes images tagged by commit SHA — never `:latest` in
  any environment's values.
- The `migrator` Job must run and succeed **before** the `api` Deployment
  rolls out to new pods (Helm hook / Argo CD sync wave) — this is what
  makes the expand/contract migration discipline in
  `docs/architecture/03-database-design.md#migrations` actually safe.
  Don't add a migration path that runs migrations from application
  startup code.
- Environment promotion is a GitOps commit (bump an image tag in the
  environments repo/folder), reviewed like any other change — never a
  manual `kubectl set image` or `helm upgrade` run by hand against
  staging/prod.
- `readinessProbe`/`livenessProbe` on every Deployment must point at
  `/readyz`/`/healthz` respectively (see
  `docs/architecture/04-api-design.md#health-and-meta-endpoints`) — a new
  deployable without both probes wired up isn't done.

## Adding a new deployable

1. Add a Helm chart under `infra/helm/{name}/` following the existing
   charts' structure (values schema, resource requests/limits set
   explicitly — no unbounded defaults).
2. Wire it into each environment's umbrella chart in
   `infra/helm/environments/{env}/`.
2. Add liveness/readiness probes.
3. Add it to the CI path-filter rules so PRs touching it get built/tested,
   and to the preview-environment (namespace-per-PR) composition if it's
   part of a reviewable user flow.
4. If it introduces a new stateful dependency, prefer an existing operator
   pattern (see the one rule above) over a new cloud-managed service.

## Observability wiring

Every new deployable must emit OpenTelemetry metrics/traces and structured
JSON logs to stdout from day one (see
`docs/architecture/08-observability.md`) — this isn't a follow-up task,
it's part of "the deployable exists." A service with no metrics/traces is
invisible to the alerting rules the rest of the platform depends on.

## When in doubt

If a task seems to need a cloud-specific feature with no portable
equivalent, that's a real trade-off worth surfacing to the user explicitly
— don't silently pick the cloud-specific path because it's easier. Check
`docs/adr/0001-cloud-agnostic-kubernetes-hosting.md`'s "Consequences"
section first; it already names the trade-off this usually is.
