# 07 — Infrastructure and Deployment

Cloud-agnostic Kubernetes hosting (see
[ADR-0001](../adr/0001-cloud-agnostic-kubernetes-hosting.md)): every
deployable runs the same way on any conformant K8s cluster — a managed
service from any provider, a self-managed cluster, or a local cluster for
development.

## Deployables

| Deployable | Kind | Notes |
|---|---|---|
| `api` | Deployment | The .NET modular monolith. Stateless; horizontally scaled. |
| `worker` | Deployment | Hangfire background job processor (same codebase/image as `api`, different entrypoint/command). |
| `migrator` | Job | Runs EF Core migrations; a pre-deploy gate, not a long-running pod. |
| `frontend-public` | Deployment | Next.js server (SSR/ISR needs a running Node process, not just static hosting). |
| `frontend-admin` | Deployment (or static hosting via the ingress) | Built static assets; served by a minimal static file server or directly by the ingress/CDN. |
| `postgresql` | CloudNativePG `Cluster` CRD | Primary + streaming replica, automated failover, WAL archiving to object storage. |
| `redis` | Redis operator `Redis`/`Sentinel` CRD | Cache, locks, rate-limit counters. |
| `minio` (dev/self-hosted) | MinIO operator `Tenant` CRD | S3-API object storage; swapped for a cloud bucket via config in managed environments, same `IMediaStorage` code path. |

Everything is packaged as **Helm charts** in `infra/helm/`, one chart per
deployable plus an umbrella chart per environment that pins versions and
environment-specific values.

## Environments

- **dev** — shared cluster namespace, latest `main` builds, relaxed
  resource limits, seeded sample content.
- **preview** — ephemeral, **namespace-per-PR**, spun up by CI for any PR
  touching `backend/`, `frontend-public/`, or `frontend-admin/`, torn down
  on merge/close. Lets reviewers click a real, running environment instead
  of reading a diff.
- **staging** — production-shaped (same Helm values class as prod, smaller
  resource requests), the GitOps promotion target before prod.
- **prod** — stricter change control: promotion requires a green staging
  deploy and passing smoke tests; rollback is a one-command GitOps revert.

## CI/CD

**GitHub Actions**, split by what changed (monorepo path filters):

1. **PR**: lint, typecheck, unit tests, and (for backend) integration
   tests against Testcontainers-provisioned Postgres/Redis — see
   [09-testing-strategy.md](09-testing-strategy.md). Build (but don't
   push) container images to catch Dockerfile breakage early.
2. **Merge to `main`**: build and push multi-arch images to the registry
   (GHCR), tagged with the commit SHA; bump the relevant Helm chart's
   image tag in the GitOps environments repo/folder for `dev`.
3. **Promotion to staging/prod**: a manual (or scheduled) GitOps commit
   bumping the image tag in that environment's values — reviewed like any
   other PR, giving a full audit trail of what's running where.

**GitOps** via Argo CD: it continuously reconciles each environment's
actual cluster state against the environments repo, so "what's deployed"
is always exactly what's committed — no `kubectl apply` from a laptop.

## Database migrations in the deploy pipeline

The `migrator` Job runs and must complete successfully **before** the new
`api` image is rolled out (Argo CD sync wave / Helm hook), enforcing the
expand/contract migration discipline from
[03-database-design.md](03-database-design.md#migrations): a migration
that isn't safe to run ahead of the new code deploying is a migration
that's structured wrong, not a reason to skip the gate.

## Rollout strategy

Rolling deployment with `readinessProbe`/`livenessProbe` wired to
`/healthz`/`/readyz` (see
[04-api-design.md](04-api-design.md#health-and-meta-endpoints)) is the
default. Argo Rollouts (canary, automated analysis against error-rate
metrics) is a Phase 5+ upgrade once there's enough production traffic for
a canary window to be statistically meaningful — see
[roadmap](../roadmap.md).

## Infrastructure as Code

**Terraform**, structured as a thin, provider-specific module
(`infra/terraform/providers/{aws,azure,gcp,on-prem}`) behind one common
interface: cluster, DNS zone, TLS cert issuer wiring, and (for managed
environments) the object storage bucket. Swapping providers means
swapping which provider module a `terraform.tfvars` points at — the
Kubernetes-level Helm charts, and everything above them, don't change.
Where a provider offers a managed Postgres/Redis, that's a documented
*option* per environment, not the default — the operator-based path (
CloudNativePG, Redis operator) is the one guaranteed to work identically
everywhere, including on-prem or a provider with no managed offering.

## Local development

- `kind` or `minikube` cluster + **Tilt** (or Skaffold) for a fast inner
  loop — file save → image rebuild → redeployed pod, with log
  aggregation, in seconds.
- A `docker-compose.yml` fallback for contributors who just need the API +
  Postgres + Redis + MinIO running without a full Kubernetes setup (e.g.
  frontend-only work).

## Ingress, TLS, CDN

- **nginx-ingress** (or Traefik) at the cluster edge; **cert-manager**
  with Let's Encrypt (or any ACME/CA) for automated TLS certificates.
- CDN is optional and provider-agnostic — any CDN pointed at the ingress
  gets the benefit of `frontend-public`'s ISR caching headers; nothing in
  the app assumes a specific CDN vendor.

## Backups and disaster recovery

Covered in detail in
[03-database-design.md](03-database-design.md#backups-and-recovery) for
the database; **Velero** additionally handles whole-namespace/persistent
volume backup and cross-cluster restore for full environment recovery or
migration between providers — the practical proof that the "cloud-agnostic"
claim holds.
