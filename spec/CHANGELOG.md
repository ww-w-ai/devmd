# CHANGELOG.md Specification

> Version: 0.1.0-draft | Status: Draft

## Purpose

CHANGELOG.md provides a structured version history with explicit breaking changes, migration paths, and deprecation timelines. It complements git history with human- and AI-readable release semantics.

## Frontmatter Schema

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Project or package name |
| `versions` | `Version[]` | REQUIRED | Release entries. Min 1. MUST be in reverse chronological order. |

### Version

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `version` | `string` | REQUIRED | SemVer version string (e.g., `1.2.0`) |
| `date` | `string` | REQUIRED | Release date in ISO 8601 format (`YYYY-MM-DD`) |
| `summary` | `string` | REQUIRED | One-line release summary |
| `breaking` | `BreakingChange[]` | OPTIONAL | Breaking changes in this release |
| `added` | `string[]` | OPTIONAL | New features or capabilities |
| `deprecated` | `Deprecation[]` | OPTIONAL | Deprecated items |
| `fixed` | `string[]` | OPTIONAL | Bug fixes |
| `schema_changes` | `SchemaChange[]` | OPTIONAL | Database or data model changes |

### BreakingChange

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `area` | `string` | REQUIRED | Affected area (e.g., `API`, `schema`, `config`) |
| `change` | `string` | REQUIRED | What changed |
| `migration` | `string` | REQUIRED | Migration instructions |
| `deadline` | `string` | OPTIONAL | Migration deadline in ISO 8601 format |

### Deprecation

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `what` | `string` | REQUIRED | Deprecated item |
| `replacement` | `string` | REQUIRED | Recommended replacement |
| `removal` | `string` | REQUIRED | Version or date when removal is planned |

### SchemaChange

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `type` | `enum(add\|remove\|rename\|modify)` | REQUIRED | Change type |
| `entity` | `string` | REQUIRED | Affected entity or table |
| `details` | `string` | REQUIRED | Description of the change |

## Body Sections

Each version MUST have a corresponding body section.

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## [version] — YYYY-MM-DD` | REQUIRED | One section per version entry. Reverse chronological order. |

Within each version section, the following subsections MAY appear:

| Subsection | Presence | Content Rules |
|------------|----------|---------------|
| `### Breaking Changes` | OPTIONAL | Detailed migration guides. |
| `### Added` | OPTIONAL | New feature descriptions. |
| `### Deprecated` | OPTIONAL | Deprecation notices with timelines. |
| `### Fixed` | OPTIONAL | Bug fix descriptions. |
| `### Schema Changes` | OPTIONAL | Data model change details. |

## Cross-References

- SHOULD reference `@SCHEMA.md` for schema change context.
- SHOULD reference `@API.md` for API-related breaking changes.
- MAY reference `@PRODUCT.md` for feature context.

## Validation Rules

1. `versions` MUST be in reverse chronological order (newest first).
2. Every `BreakingChange` MUST have a non-empty `migration` field.
3. `version` MUST be a valid SemVer string.
4. `date` MUST be a valid ISO 8601 date.
5. A MAJOR version increment SHOULD have at least one `breaking` entry.

## Conflict Detection

- `schema_changes` SHOULD be consistent with entities defined in `@SCHEMA.md`.
- API breaking changes SHOULD correspond to version changes in `@API.md#versioning`.
- Deprecation `removal` versions MUST NOT reference versions that have already been released without the removal.
