# DESIGN.md Specification

> Version: 0.1.0-draft | Status: Draft

## Purpose

DESIGN.md defines visual design tokens — colors, typography, spacing, shapes, and component styles. DevMD adopts the [Google Labs DESIGN.md](https://github.com/nicolo-ribaudo/tc39-proposal-structs) spec (`google-labs-code/design.md`) as the canonical standard for this file. This specification documents DevMD's relationship to the Google standard and any DevMD-specific extensions.

## Adoption Policy

DevMD does NOT define its own DESIGN.md schema. Projects MUST follow the `google-labs-code/design.md` specification. This file exists to:

1. Document the adopted standard within the DevMD file set.
2. Define cross-references to other DevMD files.
3. Specify conflict detection rules against the broader DevMD ecosystem.

## Frontmatter Schema

Per the Google Labs DESIGN.md spec:

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `version` | `string` | OPTIONAL | Spec version (SemVer) |
| `name` | `string` | REQUIRED | Design system name |
| `description` | `string` | OPTIONAL | Brief description of the design system |
| `colors` | `map<string, Color>` | REQUIRED | Named color tokens. Values MUST be `#hex` (3, 6, or 8 digits). |
| `typography` | `map<string, Typography>` | REQUIRED | Named typography tokens. See Typography below. |
| `rounded` | `map<string, Dimension>` | OPTIONAL | Border-radius tokens (e.g., `"sm": "4px"`) |
| `spacing` | `map<string, Dimension|number>` | OPTIONAL | Spacing scale tokens |
| `components` | `map<string, map<string, string>>` | OPTIONAL | Component-level token overrides. Values MAY use `{token}` references. |

### Typography

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `fontFamily` | `string` | REQUIRED | Font family stack |
| `fontSize` | `string` | REQUIRED | Size with unit (e.g., `"16px"`, `"1rem"`) |
| `fontWeight` | `number` | OPTIONAL | CSS font-weight value |
| `lineHeight` | `string|number` | OPTIONAL | Line height |
| `letterSpacing` | `string` | OPTIONAL | Letter spacing |

## Body Sections

Sections MUST appear in this order when present. This order follows the Google standard.

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## Overview` | OPTIONAL | Design system philosophy and principles. |
| `## Colors` | REQUIRED | Expanded color documentation. MAY include semantic aliases. |
| `## Typography` | REQUIRED | Type scale rationale and usage guidance. |
| `## Layout` | OPTIONAL | Grid system, breakpoints, containers. |
| `## Elevation & Depth` | OPTIONAL | Shadow and z-index scale. |
| `## Shapes` | OPTIONAL | Border radius and shape tokens. |
| `## Components` | OPTIONAL | Component-specific token mappings. |
| `## Do's and Don'ts` | OPTIONAL | Visual examples of correct/incorrect usage. |

## Cross-References

- SHOULD reference `@UI.md` for structural context where tokens are applied.
- SHOULD reference `@SCREENS.md` for visual proof of token application.
- SHOULD reference `@BRAND.md` for brand color and typography alignment.

## Validation Rules

1. Implementations SHOULD use `npx @anthropic-ai/design.md lint` (or equivalent Google tooling) for validation.
2. Every `colors` value MUST be a valid CSS hex color (`#RGB`, `#RRGGBB`, or `#RRGGBBAA`).
3. Every `typography` entry MUST include `fontFamily` and `fontSize`.
4. Token references in `components` using `{token}` syntax MUST resolve to a defined token.

## Conflict Detection

- `colors` named `primary`, `secondary`, `accent` SHOULD align with `@BRAND.md#colors` if both files exist.
- `typography.body.fontFamily` SHOULD match `@BRAND.md#font` if both files exist.
- Component tokens MUST NOT shadow top-level tokens with incompatible types.
