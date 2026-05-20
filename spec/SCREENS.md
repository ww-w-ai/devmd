# SCREENS.md Specification

> Version: 0.1.0-draft | Status: Draft

## Purpose

SCREENS.md provides visual reference per screen and state — the rendered truth. It bridges the gap between abstract definitions and actual appearance. DESIGN.md defines tokens, UI.md defines structure, and SCREENS.md captures the actual result as screenshots and annotations.

## Frontmatter Schema

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Project or product name |
| `screens` | `map<string, Screen>` | REQUIRED | Named screen definitions. Min 1 entry. |
| `theme_comparison` | `ThemeComparison[]` | OPTIONAL | Side-by-side theme screenshots |

### Screen

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `description` | `string` | REQUIRED | What this screen shows |
| `path` | `string` | REQUIRED | URL path (e.g., `/`, `/dashboard`, `/settings`) |
| `auth` | `enum(required\|optional\|token-based\|none)` | REQUIRED | Authentication requirement |
| `viewport` | `string[]` | OPTIONAL | Target viewports (e.g., `["desktop", "tablet", "mobile"]`) |
| `states` | `map<string, string>` | REQUIRED | State-to-screenshot map. Key: state name, Value: file path. Min 1 entry. |
| `annotations` | `Annotation[]` | OPTIONAL | Visual callouts on screenshots |
| `responsive` | `string` | OPTIONAL | Responsive behavior description |
| `dark_mode` | `string` | OPTIONAL | Dark mode screenshot file path |
| `flow_ref` | `@reference` | OPTIONAL | Reference to a flow in `@FLOWS.md` |

### Annotation

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `area` | `string` | REQUIRED | Region identifier (e.g., `"top-nav"`, `"hero"`, `"sidebar"`) |
| `note` | `string` | REQUIRED | Annotation text |

### ThemeComparison

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `screen` | `string` | REQUIRED | Screen key. MUST match a key in `screens`. |
| `light` | `string` | REQUIRED | Light theme screenshot path |
| `dark` | `string` | REQUIRED | Dark theme screenshot path |

## Body Sections

Sections MUST appear in this order when present.

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## Screens` | REQUIRED | One subsection per screen with embedded screenshots and state descriptions. |
| `## State Matrix` | OPTIONAL | Table mapping screens to their available states. |
| `## Visual Principles` | OPTIONAL | How `@DESIGN.md` tokens manifest in practice. |
| `## Responsive Breakpoints` | OPTIONAL | Breakpoint definitions and per-screen responsive notes. |

## Cross-References

- MUST reference `@DESIGN.md` for the token definitions that produce these visuals.
- MUST reference `@UI.md` for the structural definitions these screens render.
- SHOULD reference `@FLOWS.md` for the user journeys that connect screens.

## Validation Rules

1. `screens` MUST contain at least 1 entry.
2. Every `Screen.states` MUST contain at least 1 entry.
3. Every `Screen.states` value (file path) SHOULD point to an existing file.
4. Every `ThemeComparison.screen` MUST match a key in `screens`.
5. Every `Screen.path` SHOULD match a page path defined in `@UI.md`.

## Conflict Detection

- Screen `path` values SHOULD correspond to pages defined in `@UI.md`. A screen with no matching UI.md page indicates a potential orphan.
- Screen `auth` values SHOULD align with `@SECURITY.md` authentication requirements for the same path.
- `flow_ref` values MUST resolve to a valid flow in `@FLOWS.md` if that file is present.
