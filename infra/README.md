# infra

Helm charts and Terraform modules for the cloud-agnostic Kubernetes
deployment. Not yet scaffolded — see [`docs/roadmap.md`](../docs/roadmap.md)
(Phase 0) for when this happens.

Planned structure:
```
infra/
  helm/
    api/  worker/  migrator/  frontend-public/  frontend-admin/
    environments/     umbrella charts per env (dev/staging/prod)
  terraform/
    providers/
      aws/ azure/ gcp/ on-prem/
```

**The one rule**: nothing here or in application config may assume a
specific cloud provider — stateful dependencies run via Kubernetes
operators (CloudNativePG, a Redis operator, MinIO) so the same charts
deploy anywhere; provider-specific pieces live only under
`terraform/providers/{provider}/`.

Read before writing anything here:
- [`docs/architecture/07-infrastructure-and-deployment.md`](../docs/architecture/07-infrastructure-and-deployment.md)
- [`docs/adr/0001-cloud-agnostic-kubernetes-hosting.md`](../docs/adr/0001-cloud-agnostic-kubernetes-hosting.md)
- The `cleanpress-infra` Claude Code skill (`.claude/skills/`)
