---
devmd: changelog
version: "1.0"
project: Twenty CRM

format: "architectural-decisions"
note: >
  This changelog documents key architectural decisions and their evolution.
  Not a commit log — focuses on strategic changes that affect how the system
  is built and operated.

epochs:
  - name: foundation
    period: "2023-Q1 to 2023-Q4"
    theme: "Core CRM with fixed schema"
  - name: metadata
    period: "2024-Q1 to 2024-Q3"
    theme: "Metadata-driven architecture"
  - name: extensibility
    period: "2024-Q4 to 2025-Q2"
    theme: "SDK, AI, ecosystem"
  - name: scale
    period: "2025-Q3 to present"
    theme: "Performance, enterprise, AI agents"
---

# Twenty CRM — Changelog

Key architectural decisions documented as version history. Each entry explains WHAT changed, WHY, and HOW to migrate.

---

## Epoch 1: Foundation (2023)

### v0.1 — Initial Architecture

**Decision**: Monolithic NestJS backend + React frontend.

```
Before: Nothing (greenfield)
After:
  twenty-front (React + Create React App)
  twenty-server (NestJS + TypeORM + PostgreSQL)
  Monolithic schema (single public schema)
```

**Why**: Fastest path to a working CRM. Single schema simpler to reason about initially.

**Impact on DevMD files**: ARCHITECTURE.md, SCHEMA.md origins.

---

### v0.3 — Recoil for State Management

**Decision**: Adopt Recoil (over Redux, Zustand, Jotai) for global state.

**Why**: Recoil's atom-based model maps naturally to CRM's many independent UI states (filters, sorts, selection, sidebar). Atoms are co-located with features, not centralized in a store.

**Trade-off**: Recoil is less maintained than alternatives. Jotai later added for local/ephemeral state, creating a dual-state pattern. See @UI.md#state.

---

### v0.5 — Multi-Tenant via PostgreSQL Schemas

**Decision**: Each workspace gets a dedicated PostgreSQL schema instead of row-level tenant isolation.

```
Before: Single public schema with workspace_id column on every table
After:  workspace_{id} schema per tenant, public schema for shared tables
```

**Why**:
- Complete data isolation without WHERE predicates on every query
- Schema-level operations (DROP SCHEMA) for workspace deletion
- Per-workspace indexing strategies
- Cleaner backup/restore per workspace

**Trade-off**: More complex migration strategy (must apply to all schemas). Cross-workspace queries require explicit schema traversal.

**Migration guide**: Existing data migrated via script that created schemas and moved rows. One-time operation.

**Impact on DevMD files**: @SCHEMA.md#multi-tenant, @OPERATIONS.md#multi-tenant-database.

---

## Epoch 2: Metadata-Driven Architecture (2024)

### v0.10 — Metadata System Introduction

**Decision**: Replace hardcoded TypeORM entities with a metadata-driven schema system. Objects, fields, relations, and indexes defined as metadata records. GraphQL schema dynamically generated from metadata.

```
Before:
  @Entity() class Person { @Column() firstName: string; ... }
  → Fixed schema, code changes required for any field modification

After:
  ObjectMetadata → FieldMetadata → RelationMetadata → IndexMetadata
  → Schema generated at runtime from metadata records
  → Custom objects and fields without code changes
```

**Why**: CRM users need to customize their data model. Traditional ORM entities require developer intervention for every schema change. Metadata-driven approach lets users (not developers) define objects and fields at runtime.

**Impact**:
- GraphQL schema regenerated on every metadata change
- Standard objects still have base definitions but can be extended
- Custom objects get full CRUD, views, filters, API automatically

**Migration guide**: Existing TypeORM entities converted to WorkspaceEntity decorators. Migration script populated ObjectMetadata and FieldMetadata from existing entity definitions.

**Impact on DevMD files**: @SCHEMA.md (fundamental restructure), @API.md#core-api, @FLOWS.md#create-custom-object-flow.

---

### v0.12 — Custom ORM (twenty-orm)

**Decision**: Build a custom ORM layer extending TypeORM instead of using TypeORM directly.

```
Before: Direct TypeORM repositories with manual workspace schema handling
After:  twenty-orm with WorkspaceEntityManager, WorkspaceRepository, composite field handling
```

**Why**:
- TypeORM does not natively support dynamic schema switching (multi-tenant)
- Composite fields (FullName, Address, Currency) need query builder awareness
- Permission enforcement at ORM level (not just resolver level)
- Event emission on every mutation for webhooks, workflows, SSE

**Key additions over TypeORM**:
- `WorkspaceEntityManager` — Sets `search_path` per request, applies RLS predicates
- `WorkspaceRepository` — Permission-aware CRUD with composite field expansion
- Custom query builders — Handle composite field filtering and sorting
- Automatic event emission — Every create/update/delete publishes to event bus

**Migration guide**: Replaced all `getRepository()` calls with `getWorkspaceRepository()`. Custom query builders handle composite field expansion that was previously manual.

**Impact on DevMD files**: @ARCHITECTURE.md#twenty-orm, @SECURITY.md#rbac (Layer 4).

---

### v0.15 — Three-Tier GraphQL API

**Decision**: Split single GraphQL endpoint into three: Core (/api), Metadata (/metadata), Admin (/admin).

```
Before: Single /api endpoint mixing data queries and metadata mutations
After:
  /api      — Dynamic schema from metadata (record CRUD)
  /metadata — Static schema for metadata management
  /admin    — Workspace administration
```

**Why**:
- Core API schema changes on every metadata change; Metadata API is static
- Different permission models (object permissions vs settings permissions)
- Separate rate limiting and caching strategies
- Core API has workspace-scoped dynamic schema; Metadata API has global static schema

**Migration guide**: Clients using metadata operations updated their endpoint from `/api` to `/metadata`. `X-Schema-Version` header added for Core API cache invalidation.

**Impact on DevMD files**: @API.md (fundamental restructure), @ARCHITECTURE.md#three-tier.

---

### v0.18 — Composite Field System

**Decision**: Introduce composite fields that map one logical field to multiple database columns.

```
Before: name: string (single column, no structure)
After:  name: FULL_NAME → nameFirstName: string + nameLastName: string
```

**Composite types introduced**:
- `FULL_NAME` → firstName + lastName
- `ADDRESS` → street1 + street2 + city + state + postcode + country + lat + lng
- `CURRENCY` → amountMicros (bigint) + currencyCode
- `LINK` → url + label
- `EMAILS` → address + isPrimary (array)
- `PHONES` → number + isPrimary (array)
- `ACTOR` → source + workspaceMemberId + name

**Why**: CRM data is inherently structured. A "name" is always first + last. An "address" is always multi-part. Composite fields enable type-safe querying (`WHERE nameFirstName = 'Jane'`), proper sorting, and UI rendering.

**Migration guide**: Existing single-column fields migrated via script that split values and populated sub-columns. Query builders updated in twenty-orm to handle expansion.

**Impact on DevMD files**: @SCHEMA.md#composite-fields, @API.md (filter syntax), @FLOWS.md (data perspective sections).

---

## Epoch 3: Extensibility (2024-2025)

### v0.22 — Apps Ecosystem (twenty-sdk)

**Decision**: Launch a developer SDK enabling third-party apps with custom objects, fields, logic functions, skills, and agents.

```
Before: Customization only through metadata API (create objects/fields)
After:  Full SDK — defineObject, defineField, defineLogicFunction, defineSkill, defineAgent
        create-twenty-app scaffolder, twenty-cli, twenty-client-sdk
```

**Components introduced**:
- `twenty-sdk` — App definition APIs
- `twenty-client-sdk` — Type-safe GraphQL client from workspace schema
- `twenty-cli` — Developer CLI (dev, build, sync, publish)
- `create-twenty-app` — Project scaffolder

**Why**: Metadata API handles simple customization. But apps need logic (event handlers, AI skills, agents). SDK provides a structured way to extend Twenty beyond schema changes.

**Impact on DevMD files**: @AGENTS.md (complete rewrite), @HARNESS.md (new file), @PRODUCT.md#apps-sdk.

---

### v0.24 — AI Integration via Vercel AI SDK

**Decision**: Integrate AI using Vercel AI SDK 6.0 as the provider abstraction layer.

```
Before: No AI features
After:
  Multi-provider AI (OpenAI, Anthropic, Google, Azure, Bedrock, Mistral, xAI)
  defineSkill() for AI tools
  defineAgent() for autonomous AI agents
  Built-in features: enrichment, email drafting, note summarization, smart search
```

**Why**:
- Vercel AI SDK provides unified interface across 7+ providers
- Per-workspace provider choice respects data sovereignty requirements
- Skill/agent pattern enables both built-in and third-party AI capabilities
- AI accesses data through same permission layer as API (no backdoors)

**Key design decisions**:
- **No automatic fallback** between providers. One provider per workspace for billing/consistency clarity.
- **Permission-scoped data access**. AI can only see what the invoking user can see.
- **Token limits per workspace**. Admins control AI spend.
- **API keys encrypted at rest**. Never logged or exposed to frontend.

**Migration guide**: No migration needed (new feature). Workspaces must opt in via AI_SETTINGS configuration.

**Impact on DevMD files**: @HARNESS.md (new file), @AGENTS.md (skills + agents sections), @CONFIG.md (AI env vars).

---

### v0.26 — Workflow Engine

**Decision**: Build visual workflow automation (trigger → conditions → actions).

```
Before: No automation. Users manually perform repetitive tasks.
After:  Workflow builder UI. Triggers on record events or schedule.
        Actions: create/update records, send emails, call webhooks, create tasks.
```

**Why**: CRM workflows are repetitive and rule-based. "When deal closes, create follow-up task and notify Slack" should not require code.

**Impact on DevMD files**: @RUNTIME.md#workflow-engine, @FLOWS.md#workflow-execution-flow, @LIFECYCLE.md#workflow-execution.

---

## Epoch 4: Scale (2025-present)

### v0.30 — Vite 7 + SWC Migration

**Decision**: Replace Create React App with Vite 7 + SWC for frontend build.

```
Before: Create React App (webpack 5), ~45s dev start, ~90s production build
After:  Vite 7 + SWC, ~2s dev start, ~15s production build
```

**Why**: CRA was deprecated. Vite provides faster dev server (native ESM), faster builds (SWC), better plugin ecosystem.

**Migration guide**: `react-scripts` replaced with `vite`. Import aliases updated from `src/` to `~/`. Environment variables prefixed `REACT_APP_` → `VITE_`.

---

### v0.32 — tsgo for Type Checking

**Decision**: Use `tsgo` (Rust-based TypeScript compiler) for type checking alongside standard TypeScript.

```
Before: tsc --noEmit (~25s for full project)
After:  tsgo (~3s for full project)
```

**Why**: Developer feedback loop. 25s type checking interrupts flow. 3s keeps developers in context.

**Limitation**: tsgo is type-checking only (no emit). Vite + SWC handle transpilation.

---

### v0.34 — Four-Layer RBAC

**Decision**: Expand RBAC from workspace roles to four permission layers.

```
Before:
  Layer 1: Workspace role (Owner, Admin, Member, Guest)
  That's it. All-or-nothing per role.

After:
  Layer 1: Workspace role
  Layer 2: Settings permissions (DATA_MODEL, SECURITY, AI_SETTINGS, etc.)
  Layer 3: Object permissions (CREATE, READ, UPDATE, DELETE per object type)
  Layer 4: Field permissions (READ, UPDATE per field)
  + Row-level security (WHERE predicates injected by twenty-orm)
```

**Why**: Enterprise customers need granular access control. "Sales can edit Opportunities but not delete People. Marketing can read Companies but not see revenue fields."

**Migration guide**: Existing roles mapped to full permission sets. Custom roles enabled for admins to create fine-grained permission sets.

**Impact on DevMD files**: @SECURITY.md#rbac (major expansion), @FLOWS.md#record-crud-with-permissions.

---

### v0.36 — Linaria Zero-Runtime CSS

**Decision**: Adopt Linaria for zero-runtime CSS-in-JS in twenty-ui.

```
Before: styled-components (runtime CSS injection)
After:  Linaria (build-time CSS extraction, zero runtime overhead)
```

**Why**: styled-components adds ~12KB to bundle and has runtime overhead (style injection on every render). Linaria extracts CSS at build time — zero runtime cost, better performance.

**Trade-off**: Linaria has fewer features (no dynamic theme access in template literals). Theme values accessed via CSS custom properties instead.

**Impact on DevMD files**: @DESIGN.md#linaria-styling-pattern.

---

## Breaking Change Summary

| Version | Breaking Change | Migration Effort |
|---|---|---|
| v0.5 | Multi-tenant schema split | High (one-time data migration) |
| v0.10 | Metadata-driven entities | High (entity → metadata migration) |
| v0.15 | Three-tier GraphQL split | Medium (update API endpoints) |
| v0.18 | Composite fields | Medium (column split migration) |
| v0.30 | Vite migration | Low (build config changes) |
| v0.34 | Four-layer RBAC | Low (additive, existing roles preserved) |
| v0.36 | Linaria migration | Medium (styled-components → Linaria rewrite) |

## Deprecation Policy

1. Feature announced as deprecated (at least 1 minor version before removal)
2. Deprecation warning in logs and API responses
3. Breaking change detection blocks removal in PR. See @DEVOPS.md#breaking-change-detection.
4. Removed in next major version with migration guide in CHANGELOG.md
5. Migration guide includes: before/after code samples, automated migration script if possible

## Cross-References

- @ARCHITECTURE.md — Current architecture (result of all decisions)
- @SCHEMA.md — Current schema (evolved through metadata epochs)
- @API.md — Three-tier API (v0.15 decision)
- @SECURITY.md — Four-layer RBAC (v0.34 decision)
- @DESIGN.md — Linaria styling (v0.36 decision)
- @AGENTS.md — SDK and AI (v0.22 + v0.24 decisions)
- @HARNESS.md — AI integration details (v0.24 decision)
- @DEVOPS.md — Breaking change detection in CI
- @TESTING.md — How tests evolved with architecture
