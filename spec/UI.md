# UI.md Specification

> Version: 0.1.0-draft | Status: Draft

## Purpose

UI.md defines frontend structure — navigation, layouts, pages, components, forms, and interaction patterns. It is the structural companion to DESIGN.md (which defines visuals). DESIGN.md answers "how does it look?"; UI.md answers "what is where?"

## Frontmatter Schema

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Application name |
| `type` | `enum(feed\|dashboard\|landing\|saas\|e-commerce\|docs\|mobile)` | REQUIRED | Application archetype |
| `navigation` | `Navigation` | REQUIRED | See Navigation below |
| `layouts` | `map<string, Layout>` | REQUIRED | Named layout definitions. Min 1 entry. |
| `pages` | `map<string, Page>` | REQUIRED | Named page definitions. Min 1 entry. |
| `components` | `map<string, Component>` | OPTIONAL | Reusable component definitions |
| `forms` | `map<string, Form>` | OPTIONAL | Form definitions |
| `flows` | `string[]` | OPTIONAL | References to `@FLOWS.md` flow names |
| `patterns` | `Patterns` | OPTIONAL | UI behavior patterns |
| `transitions` | `Transitions` | OPTIONAL | Animation and transition rules |

### Navigation

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `primary` | `NavItem[]` | REQUIRED | Primary navigation items. Min 1 entry. |
| `footer` | `NavItem[]` | OPTIONAL | Footer navigation items |
| `mobile` | `enum(hamburger\|bottom-tab\|drawer)` | OPTIONAL | Mobile navigation pattern |

### NavItem

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `label` | `string` | REQUIRED | Display text |
| `to` | `string` | REQUIRED | Route path or URL |

### Layout

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `header` | `string\|boolean` | OPTIONAL | Header variant or `false` to hide |
| `main` | `string` | OPTIONAL | Main content area class or behavior |
| `footer` | `string\|boolean` | OPTIONAL | Footer variant or `false` to hide |
| `max_width` | `string` | OPTIONAL | Max content width (e.g., `"1200px"`) |

### Page

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `layout` | `string` | REQUIRED | MUST reference a key in `layouts` |
| `meta` | `PageMeta` | REQUIRED | Page metadata |
| `sections` | `Section[]` | REQUIRED | Ordered list of page sections. Min 1 entry. |

### PageMeta

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `title` | `string` | REQUIRED | Page title |
| `description` | `string` | REQUIRED | Meta description |

### Section

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `type` | `string` | REQUIRED | Abstraction type. See Abstraction Types below. |
| `data` | `string\|map<string, any>` | OPTIONAL | Data source or inline data |
| `behavior` | `string` | OPTIONAL | Interaction behavior description |

### Component

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `elements` | `Element[]` | REQUIRED | Component parts. Min 1 entry. |

### Element

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Element identifier |
| `source` | `string` | OPTIONAL | Data source reference |
| `format` | `string` | OPTIONAL | Display format |
| `type` | `string` | OPTIONAL | Element type hint |

### Form

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `fields` | `FormField[]` | REQUIRED | Form fields. Min 1 entry. |
| `submit` | `FormSubmit` | REQUIRED | Submit action |
| `states` | `FormStates` | OPTIONAL | UI state descriptions |

### FormField

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Field name |
| `type` | `string` | REQUIRED | Input type (e.g., `text`, `email`, `select`) |
| `required` | `boolean` | OPTIONAL | Whether field is required. Default: `false`. |

### FormSubmit

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `label` | `string` | REQUIRED | Button text |
| `endpoint` | `string` | REQUIRED | API endpoint |

### FormStates

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `idle` | `string` | OPTIONAL | Idle state description |
| `loading` | `string` | OPTIONAL | Loading state description |
| `success` | `string` | OPTIONAL | Success state description |
| `error` | `string` | OPTIONAL | Error state description |

### Patterns

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `loading` | `enum(skeleton\|spinner\|shimmer)` | OPTIONAL | Loading indicator style |
| `empty` | `enum(centered-message\|illustration\|cta)` | OPTIONAL | Empty state style |
| `error` | `enum(inline-message\|toast\|page)` | OPTIONAL | Error display style |
| `theme` | `enum(dark-only\|light-only\|system\|toggle)` | OPTIONAL | Theme mode |

### Transitions

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `page` | `string` | OPTIONAL | Page transition type (e.g., `fade`, `slide`) |
| `component` | `string` | OPTIONAL | Component transition type |
| `duration` | `string` | OPTIONAL | Default duration (e.g., `"200ms"`) |

## Abstraction Types

Each type name encodes a well-known UI pattern. AI agents SHOULD expand these into full implementations without additional specification.

| Type | Estimated Token Savings | Description |
|------|------------------------|-------------|
| `hero-tagline` | ~200 | Hero section with headline and CTA |
| `card-feed` | ~500 | Scrollable list/grid of cards |
| `tab-filter` | ~300 | Tabbed content with filter controls |
| `email-cta` | ~400 | Email capture form with CTA |
| `prose` | ~100 | Long-form text content block |
| `legal` | ~100 | Legal text (terms, privacy) |
| `sidebar` | ~400 | Side navigation or widget area |
| `data-table` | ~600 | Sortable, filterable data table |
| `dashboard-grid` | ~500 | Multi-widget dashboard layout |
| `form` | ~300 | Structured form with validation |

Custom types MAY be defined in the `components` field.

## Body Sections

Sections MUST appear in this order when present.

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## Navigation` | REQUIRED | Navigation structure rationale and responsive behavior. |
| `## Layouts` | REQUIRED | Layout descriptions and usage guidance. |
| `## Pages` | REQUIRED | Per-page documentation. One subsection per page. |
| `## Components` | OPTIONAL | Reusable component catalog. |
| `## Forms` | OPTIONAL | Form UX patterns and validation rules. |
| `## Patterns` | REQUIRED | Loading, empty, error, and theme patterns. |
| `## Constraints` | OPTIONAL | Technical or design constraints. |

## Cross-References

- MUST reference `@DESIGN.md` for visual tokens (colors, typography, spacing).
- SHOULD reference `@API.md` for data sources used in sections and forms.
- SHOULD reference `@FLOWS.md` for user journey definitions.
- SHOULD reference `@SCREENS.md` for visual verification of structure.

## Validation Rules

1. Every `Page.layout` value MUST match a key in `layouts`.
2. `navigation.primary` MUST contain at least 1 entry.
3. `pages` MUST contain at least 1 entry.
4. Every `Section.type` SHOULD be a recognized abstraction type or defined in `components`.
5. Every `FormSubmit.endpoint` SHOULD match an endpoint in `@API.md`.

## Conflict Detection

- Every layout key referenced by a page MUST exist in `layouts`.
- Every API endpoint referenced in forms or section data MUST exist in `@API.md` if that file is present.
- Navigation `to` paths SHOULD correspond to defined pages or external URLs.
