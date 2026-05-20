# ARCHITECTURE.md Specification

> Version: 0.1.0-draft | Status: Draft

## Purpose

ARCHITECTURE.md defines the system structure, package layout, layer boundaries, dependency rules, and architectural decisions.

## Frontmatter Schema

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | System or project name |
| `type` | `enum(monorepo\|polyrepo\|monolith\|microservices)` | REQUIRED | Repository/system topology |
| `packages` | `Package[]` | REQUIRED (monorepo) | Package listing. REQUIRED when `type` is `monorepo`. |
| `layers` | `Layer[]` | OPTIONAL | Architectural layers with dependency direction |

### Package

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Package name |
| `path` | `string` | REQUIRED | Relative path from project root |
| `type` | `enum(app\|library\|tool\|test\|infra)` | REQUIRED | Package classification |
| `description` | `string` | REQUIRED | What this package does |

### Layer

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Layer name (e.g., "domain", "application", "infrastructure") |
| `description` | `string` | REQUIRED | Layer responsibility |
| `depends_on` | `string[]` | OPTIONAL | Names of layers this layer MAY import from. Empty means no dependencies. |

## Body Sections

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## Overview` | REQUIRED | High-level system description. MAY include diagrams (Mermaid). |
| `## Package Map` | REQUIRED (monorepo) | REQUIRED when `type` is `monorepo`. Directory tree or table. |
| `## Layer Architecture` | REQUIRED | Layer diagram and descriptions. MUST define dependency direction. |
| `## Dependency Rules` | REQUIRED | Explicit rules for what can import what. Use MUST/MUST NOT. |
| `## Key Patterns` | OPTIONAL | Major design patterns (e.g., Repository, CQRS, Event Sourcing). |
| `## ADRs` | OPTIONAL | Architecture Decision Records. See ADR format below. |

### ADR Format

Each ADR within the `## ADRs` section MUST contain:

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `id` | `string` | REQUIRED | Unique identifier (e.g., "ADR-001") |
| `title` | `string` | REQUIRED | Decision title |
| `date` | `string` | REQUIRED | ISO 8601 date |
| `status` | `enum(accepted\|deprecated\|superseded)` | REQUIRED | Current status |
| `context` | `markdown` | REQUIRED | Why this decision was needed |
| `decision` | `markdown` | REQUIRED | What was decided |
| `consequences` | `markdown` | REQUIRED | Positive and negative outcomes |

## Cross-References

- SHOULD reference `@SCHEMA.md` for data layer alignment.
- SHOULD reference `@API.md` for interface layer alignment.
- SHOULD reference `@INFRA.md` for deployment topology.
- MAY reference `@RUNTIME.md` for agent execution context.

## Validation Rules

1. When `type` is `monorepo`, `packages` MUST contain at least 1 entry.
2. Each `Package.path` MUST be a valid relative path.
3. `Layer.depends_on` entries MUST reference layer names defined in the same `layers` array.
4. ADR `id` values MUST be unique within the file.

## Conflict Detection

- `packages` list MUST match actual directory structure. A linter SHOULD verify paths exist.
- Layer dependency rules MUST NOT create circular dependencies.
- `packages[].type` of `infra` SHOULD align with resources declared in `@INFRA.md`.
