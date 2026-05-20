---
devmd: architecture
version: "1.0"
project: documenso
updated: 2026-05-13

type: monorepo
build_system: turborepo
package_manager: npm
node_version: "22"
typescript: strict

monorepo:
  apps:
    - name: remix
      path: apps/remix
      description: "Main web application — React Router v7 + Hono server"
      port: 3000
      framework: "React Router v7 (formerly Remix)"
      server: "Hono HTTP server"
      build: "Vite"

    - name: docs
      path: apps/docs
      description: "Documentation site"

    - name: openpage-api
      path: apps/openpage-api
      description: "Public API for open page metrics"

  packages:
    - name: "@documenso/lib"
      path: packages/lib
      description: "Core business logic — 40+ server-only modules organized by domain"
      layer: domain

    - name: "@documenso/trpc"
      path: packages/trpc
      description: "tRPC router definitions — 15 routers"
      layer: api

    - name: "@documenso/api"
      path: packages/api
      description: "REST API layer (v1 deprecated, v2 tRPC-OpenAPI bridge)"
      layer: api

    - name: "@documenso/prisma"
      path: packages/prisma
      description: "Prisma schema, migrations, seed data, Kysely types"
      layer: data

    - name: "@documenso/ui"
      path: packages/ui
      description: "50+ UI primitives — Radix UI + shadcn/ui + CVA"
      layer: presentation

    - name: "@documenso/email"
      path: packages/email
      description: "Email templates and sending logic — 4 swappable providers"
      layer: infrastructure

    - name: "@documenso/auth"
      path: packages/auth
      description: "Custom auth server on Hono — email/password, OAuth, Passkeys, 2FA"
      layer: infrastructure

    - name: "@documenso/signing"
      path: packages/signing
      description: "PDF cryptographic signing — local P12 or Google Cloud HSM"
      layer: infrastructure

    - name: "@documenso/ee"
      path: packages/ee
      description: "Enterprise features — embedding, advanced templates"
      layer: domain

    - name: "@documenso/assets"
      path: packages/assets
      description: "Static assets — images, fonts"
      layer: shared

    - name: "@documenso/app-tests"
      path: packages/app-tests
      description: "Playwright E2E test suites"
      layer: testing

    - name: "@documenso/tailwind-config"
      path: packages/tailwind-config
      description: "Shared Tailwind CSS configuration"
      layer: shared

    - name: "@documenso/tsconfig"
      path: packages/tsconfig
      description: "Shared TypeScript configurations"
      layer: shared

layers:
  presentation:
    description: "React components, routes, pages"
    depends_on: [api, domain]
    packages: ["@documenso/ui"]
    apps: [remix]

  api:
    description: "tRPC routers, OpenAPI bridge, REST endpoints"
    depends_on: [domain]
    packages: ["@documenso/trpc", "@documenso/api"]

  domain:
    description: "Business logic modules — pure functions, no framework dependencies"
    depends_on: [data, infrastructure]
    packages: ["@documenso/lib", "@documenso/ee"]

  data:
    description: "Database schema, ORM, migrations"
    depends_on: []
    packages: ["@documenso/prisma"]

  infrastructure:
    description: "External service adapters — auth, email, signing, storage, jobs"
    depends_on: [data]
    packages: ["@documenso/auth", "@documenso/email", "@documenso/signing"]

dependency_direction: "presentation → api → domain → data/infrastructure"
---

# ARCHITECTURE.md — Documenso

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    apps/remix                            │
│         React Router v7 + Hono HTTP Server               │
│    ┌──────────────┬──────────────┬──────────────┐       │
│    │  Routes/Pages │  Loaders    │  Actions     │       │
│    └──────┬───────┴──────┬──────┴──────┬───────┘       │
│           │              │             │                │
│    ┌──────▼──────────────▼─────────────▼───────┐       │
│    │           Hono Server Middleware            │       │
│    │   auth │ files │ ai │ v1-rest │ v2-trpc    │       │
│    └────────────────────┬──────────────────────┘       │
└─────────────────────────┼───────────────────────────────┘
                          │
┌─────────────────────────┼───────────────────────────────┐
│    packages/trpc        │        packages/api            │
│    ┌────────────────────▼──────────────────────┐        │
│    │  15 tRPC Routers (internal + OpenAPI)      │        │
│    │  document │ template │ envelope │ ...      │        │
│    └────────────────────┬──────────────────────┘        │
└─────────────────────────┼───────────────────────────────┘
                          │
┌─────────────────────────┼───────────────────────────────┐
│    packages/lib         │                                │
│    ┌────────────────────▼──────────────────────┐        │
│    │  40+ Domain Modules (server-only)          │        │
│    │  envelope/ │ recipient/ │ field/ │ ...     │        │
│    │  signing/ │ team/ │ org/ │ webhook/ │ ...  │        │
│    └────────────────────┬──────────────────────┘        │
└─────────────────────────┼───────────────────────────────┘
                          │
         ┌────────────────┼────────────────┐
         │                │                │
┌────────▼──────┐ ┌───────▼──────┐ ┌──────▼───────┐
│ packages/     │ │ packages/    │ │ packages/    │
│ prisma        │ │ auth         │ │ signing      │
│ PostgreSQL 15 │ │ Hono auth    │ │ PDF crypto   │
│ + Kysely      │ │ server       │ │ P12/HSM      │
└───────────────┘ └──────────────┘ └──────────────┘
```

## Server Architecture

The main app (`apps/remix`) runs a **Hono HTTP server** that mounts multiple route trees:

```typescript
// Simplified Hono server setup
const app = new Hono()

// Auth routes — @documenso/auth
app.route('/auth', authRoutes)

// File upload/download
app.route('/files', fileRoutes)

// AI features
app.route('/ai', aiRoutes)

// v1 REST API (deprecated) — @documenso/api
app.route('/api/v1', v1Routes)

// v2 tRPC + OpenAPI — @documenso/trpc
app.route('/api/v2', trpcOpenAPIRoutes)

// Internal tRPC (frontend ↔ backend) — @documenso/trpc
app.route('/trpc', internalTrpcRoutes)

// React Router v7 SSR handler (catch-all)
app.use('*', reactRouterHandler)
```

## Dependency Graph

```
@documenso/ui ─────────────► (no internal deps, pure React + Radix)
     │
apps/remix ────┬──► @documenso/trpc ──► @documenso/lib ──► @documenso/prisma
               │                              │
               ├──► @documenso/auth            ├──► @documenso/signing
               │                              │
               ├──► @documenso/ui              ├──► @documenso/email
               │                              │
               └──► @documenso/assets          └──► @documenso/ee
```

## Domain Module Organization

`@documenso/lib` contains 40+ modules organized by domain concern:

```
packages/lib/
├── server-only/
│   ├── envelope/          # Create, update, send, complete envelopes
│   ├── document/          # Document CRUD, status transitions
│   ├── template/          # Template management
│   ├── recipient/         # Recipient CRUD, signing status
│   ├── field/             # Field placement, validation
│   ├── signing/           # Signature capture, PDF sealing
│   ├── team/              # Team CRUD, member management
│   ├── organisation/      # Org settings, branding
│   ├── user/              # User profile, preferences
│   ├── auth/              # Session validation, token management
│   ├── webhook/           # Webhook dispatch, retry logic
│   ├── api-token/         # API token generation, validation
│   ├── folder/            # Document folder organization
│   ├── admin/             # Admin operations
│   ├── background-job/    # Job scheduling, execution
│   ├── email/             # Email template rendering, sending
│   ├── audit-log/         # Audit trail recording
│   ├── subscription/      # Stripe billing integration
│   ├── storage/           # File storage abstraction
│   └── crypto/            # Cryptographic utilities
├── universal/             # Shared utilities (isomorphic)
│   ├── extract-request-metadata.ts
│   ├── document-audit-logs.ts
│   └── ...
└── constants/             # Shared constants and config
```

## Architecture Decision Records (ADRs)

### ADR-001: React Router v7 over Next.js

- **Decision**: Migrated from Next.js to React Router v7 (formerly Remix) with Hono server.
- **Rationale**: Better control over server middleware, streaming SSR, and Hono's lightweight composability. Avoids Next.js vendor lock-in to Vercel.

### ADR-002: tRPC + OpenAPI over REST

- **Decision**: Use tRPC for internal frontend-backend communication, with OpenAPI bridge for external API consumers.
- **Rationale**: Type-safe end-to-end with tRPC. OpenAPI export gives external developers standard REST docs without maintaining two API layers.

### ADR-003: Polymorphic Envelope model

- **Decision**: Unify Document and Template under a single `Envelope` aggregate with `EnvelopeType` discriminator.
- **Rationale**: Documents and Templates share 90% of the same data structure (recipients, fields, items). A unified model reduces code duplication.

### ADR-004: Swappable infrastructure providers

- **Decision**: Abstract storage, signing, email, and background jobs behind provider interfaces.
- **Rationale**: Self-hosters need flexibility. A solo developer uses local DB queue + filesystem; an enterprise uses BullMQ + S3 + Google HSM. See @INFRA.md#swappable-providers.

### ADR-005: Prisma + Kysely dual ORM

- **Decision**: Use Prisma for schema management and simple queries; Kysely for complex queries and raw SQL needs.
- **Rationale**: Prisma's migration and type generation are excellent, but its query builder struggles with complex joins and aggregations. Kysely fills that gap without abandoning Prisma's schema-as-code approach.

### ADR-006: Custom auth over Auth.js

- **Decision**: Build custom auth server on Hono instead of using Auth.js/NextAuth.
- **Rationale**: Need fine-grained control over session management, multi-provider support (including Passkeys and TOTP), and the auth server architecture needed to work independently of the web framework.

## Cross-References

- Database schema: @SCHEMA.md
- API layer: @API.md
- Infrastructure details: @INFRA.md
- Background jobs: @RUNTIME.md
- UI structure: @UI.md
- Auth details: @SECURITY.md#authentication
