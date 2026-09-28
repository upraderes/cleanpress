# packages

Shared TypeScript packages consumed by both `frontend-public` and
`frontend-admin`. Not yet scaffolded — see [`docs/roadmap.md`](../docs/roadmap.md)
(Phase 0) for when this happens.

Planned packages:
- `api-client` — TypeScript client generated from the backend's OpenAPI
  spec. Never hand-edit generated files.
- `ui-kit` — shared design tokens and primitive components. Must stay
  framework-neutral (no Next.js- or Vite-specific APIs).
- `config` — shared `tsconfig`, ESLint, and Prettier configuration.

See [`docs/architecture/05-frontend-architecture.md`](../docs/architecture/05-frontend-architecture.md#shared-packages-packages)
and the `cleanpress-react-conventions` Claude Code skill (`.claude/skills/`).
