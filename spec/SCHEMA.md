# SCHEMA.md Specification

> Version: 0.1.0-draft | Status: Draft

## Purpose

SCHEMA.md defines database models, fields, relationships, indexes, enums, and migration rules.

## Frontmatter Schema

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Schema name |
| `database` | `Database` | REQUIRED | Database engine info |
| `orm` | `ORM` | OPTIONAL | ORM/query builder info |
| `models` | `map<string, Model>` | REQUIRED | Keyed by PascalCase model name |
| `enums` | `map<string, string[]>` | OPTIONAL | Shared enum definitions |
| `migration_rules` | `MigrationRules` | OPTIONAL | Migration constraints |

### Database

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `type` | `enum(postgresql\|mysql\|sqlite\|mongodb\|d1\|dynamodb\|other)` | REQUIRED | Engine type |
| `version` | `string` | REQUIRED | Engine version |

### ORM

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | ORM name (e.g., "drizzle", "prisma") |
| `version` | `string` | REQUIRED | ORM version |

### Model

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `description` | `string` | REQUIRED | What this model represents |
| `fields` | `map<string, Field>` | REQUIRED | Keyed by field name |
| `indexes` | `Index[]` | OPTIONAL | Non-primary indexes |
| `enums_used` | `string[]` | OPTIONAL | References to keys in top-level `enums` |

### Field

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `type` | `string` | REQUIRED | Data type (e.g., "string", "integer", "uuid", "timestamp") |
| `required` | `boolean` | OPTIONAL | Default: `true` |
| `unique` | `boolean` | OPTIONAL | Whether a unique constraint applies |
| `default` | `any` | OPTIONAL | Default value |
| `references` | `string` | OPTIONAL | Foreign key in `@ModelName.field` format |
| `description` | `string` | OPTIONAL | Field purpose |

### Index

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `fields` | `string[]` | REQUIRED | Field names in index order |
| `unique` | `boolean` | OPTIONAL | Default: `false` |
| `type` | `string` | OPTIONAL | Index type (e.g., "btree", "gin", "hash") |

### MigrationRules

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `require_backup` | `boolean` | OPTIONAL | Whether backup is required before migration |
| `allow_destructive` | `boolean` | OPTIONAL | Whether destructive migrations are allowed |
| `naming_convention` | `string` | OPTIONAL | Migration file naming pattern |

## Body Sections

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## Models` | REQUIRED | Expanded model descriptions with field-level documentation. |
| `## Relationships` | REQUIRED | Entity-relationship descriptions. MAY include Mermaid ER diagrams. |
| `## Enums` | OPTIONAL | Expanded enum descriptions and usage guidance. |
| `## Indexes` | OPTIONAL | Index strategy rationale. |
| `## Migration Rules` | OPTIONAL | Process and safety rules for schema changes. |

## Cross-References

- Model names SHOULD match terms in `@GLOSSARY.md`.
- Models SHOULD correspond to resources exposed via `@API.md`.

## Validation Rules

1. Every `Field.references` value MUST use `@ModelName.field` format and resolve to an existing model and field.
2. `enums_used` entries MUST reference keys in the top-level `enums` map.
3. `Index.fields` entries MUST reference field names within the same model.
4. Model keys MUST be PascalCase.

## Conflict Detection

- Every model referenced by an endpoint in `@API.md` MUST exist in `models`.
- Model names SHOULD exist as terms in `@GLOSSARY.md`.
- If `@ARCHITECTURE.md` defines a data layer, models MUST reside within that layer's scope.
