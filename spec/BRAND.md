# BRAND.md Specification

> Version: 0.1.0-draft | Status: Draft

## Purpose

BRAND.md defines brand voice, tone, writing rules, and copy conventions for all user-facing content.

## Frontmatter Schema

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Brand name |
| `personality` | `string[]` | REQUIRED | Brand personality traits (e.g., "bold", "trustworthy") |
| `voice` | `Voice` | REQUIRED | See Voice below. |
| `tagline` | `string` | OPTIONAL | Brand tagline |

### Voice

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `tone` | `string` | REQUIRED | Primary tone descriptor (e.g., "confident", "warm") |
| `formality` | `enum(casual\|neutral\|formal)` | REQUIRED | Formality level |
| `humor` | `enum(none\|light\|frequent)` | REQUIRED | Humor frequency |

## Body Sections

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## Voice & Tone` | REQUIRED | Expanded voice description with before/after examples showing correct vs. incorrect tone. |
| `## Writing Rules` | REQUIRED | Do's and don'ts as bullet lists. MUST contain at least 3 rules. |
| `## Terminology` | OPTIONAL | Preferred and avoided terms. SHOULD reference `@GLOSSARY.md`. |
| `## Legal` | OPTIONAL | Required disclaimers, copyright notices, trademark usage. |
| `## Visual Identity` | OPTIONAL | Brand visual guidelines. SHOULD reference `@DESIGN.md` for tokens. |

## Cross-References

- SHOULD reference `@GLOSSARY.md` for preferred terminology.
- SHOULD reference `@DESIGN.md` for visual identity tokens.
- MAY reference `@PRODUCT.md#tagline` to ensure tagline consistency.

## Validation Rules

1. `personality` MUST contain at least 1 trait.
2. `voice.tone` MUST be a non-empty string.
3. `## Writing Rules` body section MUST contain at least 3 bullet items.

## Conflict Detection

- `voice.tone` MUST NOT contradict `@PRODUCT.md` positioning (e.g., a "playful" tone for a medical compliance product is a conflict).
- If `tagline` is defined in both `BRAND.md` and `@PRODUCT.md`, they SHOULD be identical or one SHOULD reference the other via `{PRODUCT.tagline}`.
- Terminology preferences MUST NOT contradict `@GLOSSARY.md` definitions.
