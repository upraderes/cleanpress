# ADR-0001: Cloud-agnostic Kubernetes hosting

## Status
Accepted

## Context
CleanPress needs a hosting model for a .NET API, two React frontends,
PostgreSQL, Redis, object storage, and background workers. The stakeholder
explicitly requested a cloud-agnostic approach rather than committing to a
single cloud provider, so the platform must be deployable to any major
cloud or on-prem cluster without an application rewrite.

## Decision
Host everything on **Kubernetes**, with every stateful dependency
(PostgreSQL, Redis, object storage) run via a **Kubernetes operator**
(CloudNativePG, a Redis operator, MinIO operator) rather than a managed
cloud service by default. Cloud-specific integration (DNS, managed
Postgres/Redis as an *option*, cloud object storage) is isolated behind
Terraform provider modules and the application's own storage
interface (`IMediaStorage`), never hard-coded into application logic. See
[07-infrastructure-and-deployment.md](../architecture/07-infrastructure-and-deployment.md).

## Consequences

**Positive**
- The same Helm charts and manifests deploy identically to AWS EKS, Azure
  AKS, GCP GKE, a bare-metal cluster, or a local `kind`/`minikube` cluster.
- No cloud vendor lock-in on the compute or data layer; migrating providers
  is a Terraform module swap, not an application change.
- Operators (CloudNativePG especially) provide production-grade HA,
  backup, and failover without depending on a specific cloud's managed
  offering being available or affordable in every target region.

**Negative / trade-offs**
- More operational responsibility than fully managed services (e.g., a
  managed Cloud SQL/RDS instance): the team owns operator upgrades,
  cluster capacity planning, and operator-level incident response.
- Somewhat higher baseline infrastructure complexity for a small
  deployment compared to a single-cloud PaaS approach.
- A team wanting to use a specific cloud's managed Postgres for cost or
  compliance reasons can still do so per environment (Terraform supports
  it), but it is not the default path and isn't guaranteed to be
  identically configured across environments.

## Alternatives considered
- **Single-cloud PaaS** (e.g., Azure App Service + Azure SQL/Cosmos DB, or
  AWS App Runner + RDS): simpler day-one ops, but directly contradicts the
  stated cloud-agnostic requirement and creates hard lock-in.
- **Serverless functions** (Azure Functions/AWS Lambda) for the API: poor
  fit for a stateful, connection-pooled, long-request-lifecycle CMS
  backend with background workers, and inherently cloud-specific.
