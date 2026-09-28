# 01 — System Architecture

## Component map

```
                                   ┌─────────────────────────┐
                                   │        Visitors          │
                                   └────────────┬─────────────┘
                                                │ HTTPS
                                   ┌────────────▼─────────────┐
                                   │   Ingress / Edge (CDN)    │  nginx-ingress or Traefik
                                   │   TLS via cert-manager    │  + CDN in front (any provider)
                                   └──────┬──────────────┬────┘
                     ┌─────────────────────┘              └───────────────────────┐
         ┌───────────▼────────────┐                                 ┌─────────────▼────────────┐
         │   frontend-public       │                                 │     frontend-admin        │
         │   Next.js (SSR/ISR)     │                                 │     React SPA (Vite)      │
         │   React + TypeScript    │                                 │     React + TypeScript    │
         └───────────┬────────────┘                                 └─────────────┬─────────────┘
                     │ REST (typed client, OpenAPI-generated)                    │ REST
                     └───────────────────────┬──────────────────────────────────┘
                                             │
                                ┌─────────────▼─────────────┐
                                │   .NET API (Kestrel)       │   ASP.NET Core, modular monolith
                                │   Modules:                 │   (see 02-domain-model.md)
                                │   Content · Media · Identity│
                                │   Comments · Navigation     │
                                │   Appearance · Extensibility│
                                │   Settings · Search · Audit │
                                └───┬───────┬───────┬────────┘
                                   │       │       │
                  ┌─────────────────┘       │       └───────────────────┐
       ┌───────────▼──────────┐  ┌──────────▼─────────┐   ┌─────────────▼───────────┐
       │ PostgreSQL (primary)  │  │  Redis (cache/lock) │   │  Object storage (S3 API) │
       │ CloudNativePG operator│  │  Redis operator      │   │  MinIO or cloud S3/GCS/  │
       │ schema-per-module     │  │  sessions, page cache│   │  Blob via S3 gateway     │
       └───────────┬───────────┘  └──────────────────────┘   └─────────────┬───────────┘
                   │ streaming replication (read replica for public reads)  │
       ┌───────────▼───────────┐                                          │
       │  Postgres read replica │                                          │
       └────────────────────────┘                                          │
                                                                            │
                                  ┌──────────────────────┐   ┌─────────────▼───────────┐
                                  │  Background workers    │   │  Media pipeline worker   │
                                  │  (Hangfire on Postgres)│   │  (thumbnails, transcode) │
                                  │  webhook delivery,      │   │                          │
                                  │  search indexing, email │   └──────────────────────────┘
                                  └──────────────────────┘
```

## Components

| Component | Technology | Responsibility |
|---|---|---|
| **frontend-public** | Next.js (App Router), TypeScript | Renders the public website. SSR for freshness, ISR + on-demand revalidation for performance. Owns SEO output (sitemap, meta, structured data). |
| **frontend-admin** | React + Vite, TypeScript | The back office: content editor, media library, users, settings, appearance, plugins. Pure SPA — no SEO concern. |
| **API** | ASP.NET Core (.NET), C# | Single deployable, modularized internally (see [02-domain-model.md](02-domain-model.md)). Owns all business logic, validation, and authorization. |
| **Identity** | OpenIddict module within the API (independently deployable later) | Issues OAuth2/OIDC tokens for both frontends and for server-to-server (webhooks, plugins). |
| **PostgreSQL** | CloudNativePG operator on Kubernetes | System of record. Primary for writes; streaming replica for the read-heavy public site. |
| **Redis** | Redis operator | Session/token cache, distributed locks (post editing locks), fragment/output cache. |
| **Object storage** | MinIO (self-hosted) or any S3-compatible cloud bucket | Media library files, theme/plugin assets, backups. Accessed through one internal interface (`IMediaStorage`) so the provider is swappable via config. |
| **Background workers** | Hangfire (Postgres-backed) initially | Image processing, search index updates, outbound webhook delivery, scheduled publishing, email sending. |
| **Ingress** | nginx-ingress or Traefik + cert-manager | TLS termination, routing, rate limiting at the edge. |
| **CDN** | Any (CloudFront, Cloudflare, Fastly, etc.) | Optional, provider-agnostic layer in front of ingress for static assets and cached public pages. |

## Request flow examples

**Public visitor loads a blog post**
1. CDN/ingress routes to `frontend-public`.
2. Next.js serves a statically-generated page if cached and not stale;
   otherwise renders SSR, calling the API's public read endpoints
   (`GET /api/v1/posts/{slug}`) against the **read replica** path.
3. On publish/update, the backend calls a revalidation webhook on
   `frontend-public` to invalidate the specific path (on-demand ISR).

**Editor publishes a post from the back office**
1. `frontend-admin` calls `POST /api/v1/posts/{id}/publish` (authenticated,
   bearer token from Identity) against the **primary**.
2. The Content module validates capabilities, persists the change
   transactionally (post + terms + meta), and raises a domain event
   (`PostPublished`).
3. Event handlers (in-process, via MediatR) enqueue background jobs:
   search re-index, webhook delivery to any subscribed plugins, and the
   frontend-public revalidation call.

## Why one API instead of an API-per-frontend (BFF)

A single versioned REST API serves both frontends. The back office and
front office have different query shapes (admin needs full drafts/revisions/
capabilities; the public site needs published, cache-friendly reads), but
that's handled with **different endpoints and response shapes within the
same API**, not separate backend deployables — see
[04-api-design.md](04-api-design.md). This avoids duplicated business logic
across two backends for a single-site CMS at this scale. Revisit if/when the
multi-tenant SaaS path from [ADR-0003](../adr/0003-single-site-not-multi-tenant.md)
is taken.

## Portability boundary

Everything above the object-storage/Postgres/Redis line runs identically on
any Kubernetes cluster — local (kind/minikube), any cloud managed K8s
service, or bare-metal. The only per-environment variables are: which
Terraform module provisions the cluster/DNS/certs, and whether object
storage is MinIO or a cloud bucket (same S3 API either way). See
[07-infrastructure-and-deployment.md](07-infrastructure-and-deployment.md).
