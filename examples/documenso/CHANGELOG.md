---
devmd: changelog
version: "1.0"
project: documenso
updated: 2026-05-13

format: architectural-decisions
entries:
  - version: "2.0"
    title: "React Router v7 + Hono migration"
    breaking: true
  - version: "1.5"
    title: "API v2 (tRPC OpenAPI)"
    breaking: true
  - version: "1.3"
    title: "Envelope model (polymorphic unification)"
    breaking: true
---

# CHANGELOG.md — Documenso

Architectural decisions and breaking changes. For feature-level changes, see GitHub Releases.

---

## v2.0 — React Router v7 + Hono Migration

**Date**: 2025-Q4
**Breaking**: Yes
**Scope**: Full frontend + server runtime

### Decision

Migrated from **Next.js (App Router)** to **React Router v7 + Hono** as the application framework.

### Why

| Factor | Next.js | React Router v7 + Hono |
|---|---|---|
| Server runtime | Locked to Next.js server | Any Node.js HTTP server (Hono) |
| Auth flexibility | NextAuth adapter constraints | Custom auth server with full control |
| Hosting freedom | Optimized for Vercel | Runs anywhere Node.js runs |
| Build tool | Webpack/Turbopack (opinionated) | Vite (faster, more configurable) |
| Route conventions | Next.js-specific file routing | Standard React Router file routing |
| Bundle control | Next.js managed | Full control via Vite config |

### What Changed

| Before (Next.js) | After (React Router v7 + Hono) |
|---|---|
| `app/` directory with Next.js conventions | `apps/remix/app/routes/` with React Router v7 |
| `next.config.js` | `vite.config.ts` |
| NextAuth for authentication | Custom Hono auth server (`@documenso/auth`) |
| API routes in `app/api/` | Hono routes mounted on server |
| `getServerSession()` | Custom session middleware on Hono context |
| Vercel-optimized deployment | Docker-based deployment (any platform) |
| `NEXT_PUBLIC_*` / `NEXT_PRIVATE_*` env prefix | Same prefixes retained for backward compatibility |

### Migration Guide

**For self-hosters upgrading from v1.x (Next.js)**:

1. **Docker image**: Replace entirely. The new image uses `node build/server/index.js` instead of `next start`.
2. **Environment variables**: All `NEXT_PUBLIC_*` and `NEXT_PRIVATE_*` variables remain the same. No env var changes needed.
3. **Reverse proxy**: Port remains 3000. No URL path changes. Existing nginx/caddy configs work unchanged.
4. **Database**: Run `npx prisma migrate deploy`. No manual SQL needed.
5. **Custom themes**: If using custom CSS, the class names changed (Next.js-specific classes removed). Re-test UI customizations.
6. **Embedding**: Enterprise embedding iframe URLs changed from `/embed/sign/*` to same path but different internal routing. Test embedded flows.

**For API consumers**:

- No API changes. Both tRPC and OpenAPI endpoints remain at same paths.
- Session cookie name unchanged (`__session`).
- API token format unchanged (`dt_*`).

---

## v1.5 — API v2 (tRPC OpenAPI)

**Date**: 2025-Q2
**Breaking**: Yes (v1 API deprecated)
**Scope**: Public REST API

### Decision

Deprecated the **v1 REST API** (built with `ts-rest`) in favor of **v2** (auto-generated from tRPC routers via `trpc-to-openapi`).

### Why

| Factor | v1 (ts-rest) | v2 (tRPC OpenAPI) |
|---|---|---|
| Source of truth | Separate route definitions | tRPC routers (shared with frontend) |
| Type safety | Manual schema sync | Automatic from tRPC procedures |
| Maintenance | Duplicate code for every endpoint | Zero duplication — single router serves both internal + public |
| OpenAPI spec | Manually maintained | Auto-generated from tRPC meta |
| Endpoint coverage | Subset of features | Full feature parity with frontend |

### What Changed

| Before (v1) | After (v2) |
|---|---|
| `GET /api/v1/documents` | `GET /api/v2/document` |
| `POST /api/v1/documents` | `POST /api/v2/document` |
| `POST /api/v1/documents/:id/send` | `POST /api/v2/document/:id/send` |
| `ts-rest` contract definitions | `trpc-to-openapi` meta on procedures |
| Separate API package | Thin bridge in `@documenso/api` |

### Migration Guide

**For API consumers**:

1. **Base path**: Change `/api/v1/` to `/api/v2/` in all requests.
2. **Resource names**: Changed from plural to singular in some cases (e.g., `documents` → `document`). Check OpenAPI spec at `/api/v2/openapi.json`.
3. **Response format**: Wrapped in tRPC result envelope: `{ "result": { "data": { ... } } }` instead of flat response body.
4. **Error format**: Error responses now include tRPC error codes alongside HTTP status. See @ERRORS.md.
5. **Authentication**: Unchanged. Bearer token with `dt_*` prefix.
6. **Pagination**: Response shape changed. `totalPages`, `currentPage`, `perPage` fields added. See @API.md#pagination.

**v1 deprecation timeline**:

- v1.5: v2 released alongside v1. v1 marked deprecated in docs.
- v1.7: v1 endpoints return `Deprecation` header.
- v2.0: v1 endpoints removed.

---

## v1.3 — Envelope Model (Polymorphic Unification)

**Date**: 2025-Q1
**Breaking**: Yes (database schema)
**Scope**: Core data model

### Decision

Unified the previously separate `Document` and `Template` models into a single polymorphic **`Envelope`** model with `envelopeType` discriminator.

### Why

| Factor | Before (separate models) | After (Envelope) |
|---|---|---|
| Code duplication | Separate CRUD for Document and Template | Single set of operations with type filter |
| Shared features | Had to implement twice (folders, recipients, fields) | Implement once, works for both |
| Template → Document | Complex copy logic between different schemas | Clone within same table, change `envelopeType` |
| API surface | Separate routers, separate permissions | Unified router with type-aware queries |
| Future types | Would need a third model | Add new `EnvelopeType` enum value |

### What Changed

**Database**:

```sql
-- Before: two tables
Document (id, title, status, userId, teamId, ...)
Template (id, title, templateType, userId, teamId, ...)

-- After: one table
Envelope (id, envelopeType, title, status, teamId, ...)
-- envelopeType: DOCUMENT | TEMPLATE
```

**Schema changes**:

| Before | After |
|---|---|
| `Document` model | `Envelope` with `envelopeType = DOCUMENT` |
| `Template` model | `Envelope` with `envelopeType = TEMPLATE` |
| `DocumentData` (separate) | `DocumentData` (unchanged, linked to Envelope) |
| `Recipient.documentId` | `Recipient.envelopeId` |
| `Field.documentId` | `Field.envelopeId` |
| `DocumentAuditLog.documentId` | `DocumentAuditLog.envelopeId` |

**API changes**:

| Before | After |
|---|---|
| `trpc.document.*` | `trpc.document.*` (unchanged, filters `envelopeType = DOCUMENT`) |
| `trpc.template.*` | `trpc.template.*` (unchanged, filters `envelopeType = TEMPLATE`) |
| (none) | `trpc.envelope.*` (new unified queries) |

### Migration Guide

**Database migration** (handled automatically by Prisma):

1. New `Envelope` table created
2. Data copied from `Document` → `Envelope` with `envelopeType = DOCUMENT`
3. Data copied from `Template` → `Envelope` with `envelopeType = TEMPLATE`
4. Foreign keys updated on all referencing tables
5. Old `Document` and `Template` tables dropped

**For self-hosters**:

1. **Backup database** before upgrading (mandatory)
2. Run `npx prisma migrate deploy` — migration handles everything
3. Migration is irreversible. Cannot downgrade to pre-1.3 after this point
4. Estimated migration time: < 5 minutes for databases under 100K documents

**For API consumers**:

- `document.*` and `template.*` endpoints unchanged in behavior
- Internal IDs remain the same (mapped during migration)
- New `envelope.*` endpoints available for unified queries
- Webhook payloads now include `envelopeType` field

---

## Decision Log (Non-Breaking)

Minor architectural decisions that did not require breaking changes.

| Date | Decision | Rationale |
|---|---|---|
| 2025-Q4 | Biome replaces ESLint + Prettier | Single tool for lint + format. Faster. Simpler config. See @CLAUDE.md#formatting |
| 2025-Q3 | Lingui replaces next-intl | Framework-agnostic i18n. Works with React Router v7. 11 languages. See @UI.md#internationalization |
| 2025-Q3 | Konva for PDF rendering | HTML5 Canvas performance for field placement. Smooth drag-and-drop. See @DESIGN.md#pdf-canvas-konva |
| 2025-Q2 | Arctic for OAuth | Lightweight OAuth 2.0 client. Replaces NextAuth provider system. See @SECURITY.md#oauth |
| 2025-Q1 | Kysely alongside Prisma | Complex queries (joins, CTEs) that Prisma cannot express. See @SCHEMA.md |
| 2024-Q4 | 4-provider abstraction | Storage, signing, email, jobs all swappable. See @INFRA.md#swappable-providers |
| 2024-Q3 | Turborepo build system | Monorepo task orchestration with caching. See @ARCHITECTURE.md |

## Cross-References

- Current architecture: @ARCHITECTURE.md
- Current API structure: @API.md
- Current schema: @SCHEMA.md
- Deployment procedures: @DEVOPS.md
- Environment variables (unchanged across migrations): @CONFIG.md
