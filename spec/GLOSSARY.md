# GLOSSARY.md Specification

> Version: 0.1.0-draft | Status: Draft

## Purpose

GLOSSARY.md defines the canonical domain vocabulary (DDD ubiquitous language) for the project.

## Frontmatter Schema

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Glossary title |
| `default_language` | `string` | OPTIONAL | ISO 639-1 code. Default: `"en"`. |
| `terms` | `map<string, Term>` | REQUIRED | Keyed by term name (lowercase, kebab-case). |

### Term

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `definition` | `string` | REQUIRED | Concise definition. MUST NOT be empty. |
| `synonyms` | `string[]` | OPTIONAL | Alternate names for the same concept |
| `antonyms` | `string[]` | OPTIONAL | Opposite or contrasting terms |
| `context` | `string` | OPTIONAL | When or where this term applies |
| `see_also` | `string[]` | OPTIONAL | Related term keys within this glossary |
| `bounded_context` | `string` | OPTIONAL | DDD bounded context name |

## Body Sections

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## Terms` | REQUIRED | Expanded definitions with usage examples. Each term SHOULD include at least one example sentence. |
| `## Bounded Contexts` | OPTIONAL | Context map showing how bounded contexts relate. MAY use Mermaid diagrams. |
| `## Anti-Glossary` | OPTIONAL | Terms explicitly NOT used in this project, with rationale for exclusion. |

## Cross-References

- ALL other DevMD files SHOULD use terms defined here consistently.
- `see_also` entries MUST reference keys that exist within the same `terms` map.
- `bounded_context` values SHOULD be consistent across all terms.

## Validation Rules

1. Each term MUST have a non-empty `definition` field.
2. Term keys MUST be lowercase. SHOULD be kebab-case for multi-word terms.
3. `see_also` entries MUST resolve to existing keys in `terms`.
4. `default_language` MUST be a valid ISO 639-1 code if provided.

## Conflict Detection

- Every domain term used as a model name in `@SCHEMA.md` MUST appear as a key in `terms`.
- Every domain term used in `@PRODUCT.md#problem` or `@PRODUCT.md#solution` SHOULD appear in `terms`.
- If `@BRAND.md` has a Terminology section, preferred terms MUST NOT contradict definitions here.
