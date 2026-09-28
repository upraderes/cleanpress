# 08 — Observability

Cloud-agnostic by the same logic as hosting (see
[ADR-0001](../adr/0001-cloud-agnostic-kubernetes-hosting.md)): the default
stack is open-source and runs on any Kubernetes cluster; a managed
cloud-native option (Azure Monitor, CloudWatch, Cloud Logging) can be wired
in per environment as an additional sink, but nothing in the application
assumes one.

## The three pillars

| Signal | Emitted via | Stored/queried in |
|---|---|---|
| **Metrics** | OpenTelemetry SDK (.NET) / `@opentelemetry/api` (Next.js), Prometheus exposition format | Prometheus, dashboarded in Grafana |
| **Logs** | Serilog (structured JSON) in .NET, Pino in Next.js, both to stdout (Kubernetes-native) | Loki (log aggregation matched to Grafana) |
| **Traces** | OpenTelemetry SDK, W3C trace-context propagated across `frontend-public`/`frontend-admin` → API → Postgres/Redis calls | Tempo or Jaeger |

All three correlate on a shared `trace_id`, so a slow public-page load can
be traced from the Next.js request through the API call down to the
specific Postgres query.

## What gets measured

- **Golden signals** per service: request rate, error rate, p50/p95/p99
  latency — for the API overall and broken out per module (Content, Media,
  Identity, …) via a route-tag on each metric.
- **Business/content metrics**: posts published per day, comment moderation
  queue depth, media storage used, webhook delivery failure rate — surfaced
  on the back office's Site-Health-style status page (see
  [04-api-design.md](04-api-design.md#health-and-meta-endpoints)), not just
  in an ops dashboard.
- **Infrastructure**: standard Kubernetes/node metrics (CPU, memory, pod
  restarts), Postgres metrics via the CloudNativePG operator's built-in
  exporter, Redis metrics via `redis_exporter`.
- **Frontend**: Core Web Vitals (LCP, INP, CLS) collected from real users on
  `frontend-public` via the Web Vitals API, reported back to the metrics
  pipeline — SEO/performance is a monitored product requirement, not a
  one-time Lighthouse check (see
  [09-testing-strategy.md](09-testing-strategy.md)).

## Alerting

Prometheus Alertmanager, routed to Slack/email/a generic webhook (no
vendor lock to a specific on-call tool). Baseline alerts from Phase 0:
API error-rate spike, API p95 latency breach, pod crash-looping, Postgres
replication lag, disk/storage usage threshold, webhook dead-letter queue
growth, TLS certificate nearing expiry.

## Error tracking

Sentry (self-hostable, matching the cloud-agnostic stance) for exception
aggregation and stack-trace-level debugging across all four runtimes
(.NET API, worker, both frontends) — complementary to metrics/logs, not a
replacement: metrics say *something* is wrong, Sentry says *what line*.

## SLOs (initial targets, revisited once there's real traffic)

| SLO | Target |
|---|---|
| API availability | 99.9% |
| API p95 latency (read endpoints) | < 300ms |
| Public page TTFB (cached) | < 200ms |
| Public page LCP | < 2.5s |
| Background job (webhook delivery) latency | < 60s from trigger to first attempt |

## Health checks

`/healthz` and `/readyz` (see
[04-api-design.md](04-api-design.md#health-and-meta-endpoints)) drive
Kubernetes liveness/readiness probes directly — a pod failing `/readyz`
(e.g. can't reach Postgres) is pulled from the service's load balancing
without being killed; a pod failing `/healthz` (deadlocked process) is
restarted.

## Synthetic monitoring

A scheduled job (Phase 5+, see [roadmap](../roadmap.md)) hitting the public
site's homepage and a representative post page from outside the cluster,
alerting on failure/latency independent of whether internal metrics are
even being scraped correctly — catches the "everything looks green
internally but the site is actually down" class of failure (DNS, CDN,
ingress misconfiguration).
