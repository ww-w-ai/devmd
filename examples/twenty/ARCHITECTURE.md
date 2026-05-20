---
devmd: architecture
version: "1.0"
project: Twenty CRM
pattern: modular-monolith
monorepo: nx
package_manager: yarn-4
language: TypeScript 5.9
node: "24+"

layers:
  - name: presentation
    packages: [twenty-front, twenty-website-new, twenty-docs]
    description: User-facing applications
  - name: api
    packages: [twenty-server]
    description: "Three GraphQL endpoints + REST + WebSocket"
  - name: engine
    packages: [twenty-server/engine]
    description: Cross-cutting infrastructure (ORM, cache, guards, workspace)
  - name: domain
    packages: [twenty-server/modules]
    description: Business logic organized by domain
  - name: shared
    packages: [twenty-shared, twenty-utils]
    description: Cross-package types, validators, constants
  - name: sdk
    packages: [twenty-sdk, twenty-client-sdk, twenty-cli, create-twenty-app]
    description: Developer tools and extension points

dependency_direction: presentation → api → engine → domain → shared

packages:
  apps:
    - name: twenty-front
      tech: "React 18, Vite 7, SWC"
      role: Main CRM single-page application
    - name: twenty-server
      tech: "NestJS 11"
      role: "API server, Worker, CLI — 3 entry points"
    - name: twenty-docs
      tech: Documentation site
      role: Developer documentation
    - name: twenty-website-new
      tech: "Next.js"
      role: Marketing website

  libraries:
    - name: twenty-ui
      role: "Design system — components, theme, icons"
    - name: twenty-shared
      role: "14 export domains — types, validators, utils shared across apps"
    - name: twenty-utils
      role: General-purpose utilities
    - name: twenty-emails
      role: Email templates (React Email)
    - name: twenty-sdk
      role: "App SDK — defineObject, defineField, defineSkill, defineAgent"
    - name: twenty-client-sdk
      role: Type-safe GraphQL client generation
    - name: twenty-cli
      role: "CLI tool — dev, build, sync commands"
    - name: create-twenty-app
      role: App scaffolder (npx create-twenty-app)
    - name: twenty-zapier
      role: Zapier integration
    - name: twenty-claude-skills
      role: Claude Code AI skills for Twenty
    - name: twenty-e2e-testing
      role: Playwright E2E test infrastructure
    - name: twenty-oxlint-rules
      role: Custom Oxlint rules for code quality
    - name: twenty-docker
      role: Docker build configurations

adrs:
  - id: ADR-001
    title: "Metadata-driven schema over static migrations"
    status: accepted
    context: >
      CRM users need to create custom objects/fields without developer
      intervention. Static TypeORM migrations cannot support runtime schema changes.
    decision: >
      Implement a metadata layer (ObjectMetadata, FieldMetadata) that drives
      schema generation. Workspace schemas are created/modified at runtime.
      Custom ORM (twenty-orm) reads metadata to build queries.
    consequence: >
      More complex ORM layer but infinite extensibility. Users can create
      objects via UI or API without code changes.

  - id: ADR-002
    title: "Three-tier GraphQL over single endpoint"
    status: accepted
    context: >
      Core data API, metadata management API, and admin panel API have
      different auth requirements and schema structures.
    decision: >
      Separate endpoints: /api (Core — workspace data), /metadata (schema
      management), /admin (admin panel). Each has its own schema builder
      and resolvers.
    consequence: >
      Clear separation of concerns. Core API schema is dynamically generated
      from metadata. Metadata API is static NestJS resolvers.

  - id: ADR-003
    title: "Multi-tenant via PostgreSQL schemas"
    status: accepted
    context: >
      Need data isolation between workspaces. Options: row-level (shared tables),
      schema-level (separate schemas), database-level (separate DBs).
    decision: >
      Schema-per-workspace. Each workspace gets workspace_{id} schema.
      Core tables (users, workspaces, billing) in public schema.
    consequence: >
      Strong isolation without operational overhead of separate databases.
      Migration complexity for schema changes across all workspaces.

  - id: ADR-004
    title: "Custom ORM (twenty-orm) over raw TypeORM"
    status: accepted
    context: >
      TypeORM does not support metadata-driven schemas, composite fields,
      permission-aware queries, or event emission natively.
    decision: >
      Extend TypeORM with WorkspaceEntityManager, WorkspaceRepository,
      custom query builders, composite field resolution, and automatic
      event emission on mutations.
    consequence: >
      Full control over query lifecycle. RBAC enforced at ORM level.
      Higher maintenance burden for ORM layer.

  - id: ADR-005
    title: "Recoil + Jotai + Apollo Client for state management"
    status: accepted
    context: >
      Frontend needs global app state (auth, workspace, settings), local
      component state (modals, dropdowns), and server-synced state (records,
      views).
    decision: >
      Recoil for global state, Jotai for local/component state, Apollo
      Client cache for server state. No Redux.
    consequence: >
      Fine-grained reactivity. Learning curve for three state systems.
      Apollo cache serves as single source of truth for server data.
---

# Twenty CRM Architecture

## High-Level Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        Nx Monorepo                          │
│                      (20 packages)                          │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ twenty-front  │  │ twenty-docs  │  │twenty-website-new│  │
│  │ React 18+Vite │  │              │  │   Next.js        │  │
│  └──────┬───────┘  └──────────────┘  └──────────────────┘  │
│         │ GraphQL + REST                                    │
│  ┌──────▼───────────────────────────────────────────────┐   │
│  │                 twenty-server (NestJS 11)             │   │
│  │  ┌──────────┐  ┌───────────┐  ┌──────────────────┐   │   │
│  │  │ API      │  │  Worker   │  │    CLI            │   │   │
│  │  │ Server   │  │ (BullMQ)  │  │                   │   │   │
│  │  └────┬─────┘  └─────┬─────┘  └──────────────────┘   │   │
│  │       │              │                                 │   │
│  │  ┌────▼──────────────▼────────────────────────────┐   │   │
│  │  │            Engine Layer                         │   │   │
│  │  │  twenty-orm │ guards │ cache │ workspace-mgr   │   │   │
│  │  └────┬───────────────────────────────────────────┘   │   │
│  │       │                                               │   │
│  │  ┌────▼───────────────────────────────────────────┐   │   │
│  │  │          Domain Modules (70+ core, 60+ meta)   │   │   │
│  │  │  person │ company │ opportunity │ workflow │ …  │   │   │
│  │  └────┬───────────────────────────────────────────┘   │   │
│  │       │                                               │   │
│  └───────┼───────────────────────────────────────────────┘   │
│          │                                                    │
│  ┌───────▼────────────────────────────────────────────────┐  │
│  │  twenty-shared (14 domains) │ twenty-ui │ twenty-utils  │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  twenty-sdk │ twenty-client-sdk │ twenty-cli │ zapier   │  │
│  └────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
         │                    │                │
    ┌────▼────┐          ┌───▼───┐       ┌────▼────┐
    │PostgreSQL│          │ Redis │       │ClickHouse│
    │   16     │          │       │       │(analytics)│
    └─────────┘          └───────┘       └──────────┘
```

## Nx Monorepo Structure

```
twenty/
├── packages/
│   ├── twenty-front/          # React SPA (CRM UI)
│   ├── twenty-server/         # NestJS backend
│   │   ├── src/
│   │   │   ├── engine/        # Cross-cutting infrastructure
│   │   │   │   ├── core-modules/
│   │   │   │   │   ├── auth/
│   │   │   │   │   ├── billing/
│   │   │   │   │   ├── feature-flag/
│   │   │   │   │   ├── health/
│   │   │   │   │   ├── open-api/
│   │   │   │   │   ├── workspace/
│   │   │   │   │   └── ... (70+ modules)
│   │   │   │   ├── metadata-modules/
│   │   │   │   │   ├── object-metadata/
│   │   │   │   │   ├── field-metadata/
│   │   │   │   │   ├── relation-metadata/
│   │   │   │   │   ├── index-metadata/
│   │   │   │   │   ├── view-metadata/
│   │   │   │   │   └── ... (60+ modules)
│   │   │   │   ├── twenty-orm/
│   │   │   │   │   ├── workspace-entity-manager.ts
│   │   │   │   │   ├── workspace-repository.ts
│   │   │   │   │   ├── query-builders/
│   │   │   │   │   │   ├── select-query-builder.ts
│   │   │   │   │   │   ├── insert-query-builder.ts
│   │   │   │   │   │   ├── update-query-builder.ts
│   │   │   │   │   │   ├── delete-query-builder.ts
│   │   │   │   │   │   └── soft-delete-query-builder.ts
│   │   │   │   │   └── composite-field-handler.ts
│   │   │   │   ├── core-entity-cache/
│   │   │   │   ├── dataloaders/
│   │   │   │   ├── guards/
│   │   │   │   ├── middlewares/
│   │   │   │   ├── workspace-manager/
│   │   │   │   └── workspace-cache/
│   │   │   ├── modules/       # Domain modules
│   │   │   │   ├── person/
│   │   │   │   ├── company/
│   │   │   │   ├── opportunity/
│   │   │   │   ├── note/
│   │   │   │   ├── task/
│   │   │   │   ├── attachment/
│   │   │   │   ├── workflow/
│   │   │   │   ├── connected-account/
│   │   │   │   ├── messaging/
│   │   │   │   ├── calendar/
│   │   │   │   ├── dashboard/
│   │   │   │   ├── view/
│   │   │   │   ├── favorite/
│   │   │   │   ├── blocklist/
│   │   │   │   ├── webhook/
│   │   │   │   └── ... (17 domain modules)
│   │   │   └── main.ts        # 3 entry points: api, worker, cli
│   │   └── test/
│   ├── twenty-ui/             # Design system
│   ├── twenty-shared/         # Shared types & validators
│   ├── twenty-utils/          # General utilities
│   ├── twenty-emails/         # Email templates
│   ├── twenty-sdk/            # App development SDK
│   ├── twenty-client-sdk/     # Type-safe GraphQL client
│   ├── twenty-cli/            # CLI tool
│   ├── create-twenty-app/     # App scaffolder
│   ├── twenty-zapier/         # Zapier integration
│   ├── twenty-claude-skills/  # Claude Code skills
│   ├── twenty-e2e-testing/    # Playwright test infra
│   ├── twenty-oxlint-rules/   # Custom lint rules
│   ├── twenty-docker/         # Docker configurations
│   └── twenty-docs/           # Documentation site
├── nx.json
├── package.json               # Yarn 4 workspaces
└── tsconfig.base.json
```

## Engine Layer

The engine layer provides cross-cutting infrastructure shared by all domain modules. See @SCHEMA.md for data model details.

### twenty-orm

Custom ORM extending TypeORM with workspace awareness. See @GLOSSARY.md#twenty-orm.

**Key components:**
- **WorkspaceEntityManager** — Permission-aware entity manager. Checks RBAC permissions before every query. Applies row-level security predicates (WHERE clauses) based on user role.
- **WorkspaceRepository** — Repository pattern with automatic workspace schema routing. Queries go to `workspace_{id}` schema.
- **Query Builders** — Select, Insert, Update, Delete, SoftDelete. Handle composite field expansion (FullName → nameFirstName, nameLastName).
- **Event Emission** — Every mutation (create, update, delete) emits domain events for workflow triggers, webhooks, and audit logging.

### Guards & Middlewares

- **WorkspaceAuthGuard** — Validates JWT, extracts workspace context, sets current user
- **SettingsPermissionGuard** — Checks settings-level permissions (WORKSPACE, ROLES, SECURITY, DATA_MODEL, etc.)
- **NoPermissionGuard** — Blocks access to restricted endpoints
- **SchemaVersionMiddleware** — Validates X-Schema-Version header for optimistic concurrency
- **LocaleMiddleware** — Extracts x-locale header for i18n

### Workspace Manager

Manages workspace lifecycle: creation, schema provisioning, migration, deletion. When a new workspace is created, the manager:
1. Creates `workspace_{id}` PostgreSQL schema
2. Creates tables for all standard objects
3. Applies default field metadata
4. Seeds default views, pipeline stages, and settings

### Workspace Cache

Two-level cache: in-memory (per-process) + Redis (shared). Caches:
- Object metadata (schema definitions)
- Field metadata
- Workspace settings
- Feature flags

Cache invalidation on metadata changes via Redis pub/sub.

## Domain Module Pattern

Each domain module follows this structure:

```
modules/person/
├── person.module.ts           # NestJS module definition
├── person.resolver.ts         # GraphQL resolver (auto-generated from metadata)
├── person.service.ts          # Business logic
├── person.workspace-entity.ts # twenty-orm entity definition
├── person.seed.ts             # Seed data for new workspaces
├── dtos/
│   ├── create-person.input.ts
│   └── update-person.input.ts
├── listeners/
│   └── person-created.listener.ts
└── __tests__/
    ├── person.resolver.spec.ts
    └── person.service.spec.ts
```

## Metadata-Driven Schema Generation

The core architectural pattern. See @SCHEMA.md for full details.

```
ObjectMetadata + FieldMetadata + RelationMetadata
           │
           ▼
WorkspaceSchemaBuilder (NestJS service)
           │
           ▼
Dynamic GraphQL Schema (Core API)
           │
           ▼
Apollo Client queries workspace data
```

The WorkspaceSchemaBuilder reads metadata at startup and on metadata changes, generating:
- GraphQL types, inputs, queries, mutations for each object
- Filter types (StringFilter, NumberFilter, DateFilter, etc.)
- Sort types
- Connection types (cursor-based pagination)
- Aggregate types

## Cross-References

- @SCHEMA.md — Data model, metadata system, composite fields
- @API.md — Three-tier GraphQL, REST, webhooks
- @INFRA.md — Deployment topology, Docker, Kubernetes
- @SECURITY.md — Auth, RBAC, permission enforcement
- @RUNTIME.md — Worker, cron, real-time events
- @TESTING.md — Test architecture and conventions
