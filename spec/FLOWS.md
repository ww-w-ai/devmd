# FLOWS.md Specification

> Version: 0.1.0-draft | Status: Draft

## Purpose

FLOWS.md defines end-to-end UX journeys paired with backend data flows. It is the dynamic complement to UI.md (which defines static structure). Each flow captures what the user sees, what they do, what the system does in response, and how errors are handled.

## Frontmatter Schema

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Project or product name |
| `flows` | `map<string, Flow>` | REQUIRED | Named flow definitions. Min 1 entry. |

### Flow

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `trigger` | `string` | REQUIRED | What starts this flow (e.g., `"user clicks Sign Up"`) |
| `description` | `string` | OPTIONAL | Brief summary of the flow's purpose |
| `ux` | `UXStep[]` | REQUIRED | Ordered user-facing steps. Min 1 entry. |
| `data` | `DataStep[]` | REQUIRED | Ordered system-side steps. Min 1 entry. |
| `error_paths` | `ErrorPath[]` | OPTIONAL | Error scenarios and recovery actions |
| `references` | `@reference[]` | OPTIONAL | Related DevMD file references |

### UXStep

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `screen` | `string` | REQUIRED | Screen or page identifier. SHOULD match a key in `@UI.md#pages` or `@SCREENS.md#screens`. |
| `action` | `string` | REQUIRED | What the user does on this screen |

### DataStep

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `service` | `string` | REQUIRED | Service or API identifier. SHOULD match an endpoint in `@API.md` or an agent in `@RUNTIME.md`. |
| `action` | `string` | REQUIRED | What the system does (e.g., `"validate input"`, `"create record"`) |

### ErrorPath

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `at` | `string` | REQUIRED | Step identifier where the error occurs |
| `error` | `string` | REQUIRED | Error condition description |
| `recovery` | `string` | REQUIRED | How the system or user recovers |

## Body Sections

Sections MUST appear in this order when present.

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## Flows` | REQUIRED | One subsection per flow. Each subsection documents the flow's steps, decision points, and outcomes. |
| `## Flow Dependencies` | OPTIONAL | Which flows trigger or depend on other flows. MAY use a dependency table or diagram. |
| `## Error Recovery` | OPTIONAL | Cross-cutting error handling strategies that apply to multiple flows. |

## Cross-References

- MUST reference `@UI.md` for screen and page identifiers used in `ux` steps.
- MUST reference `@API.md` for service and endpoint identifiers used in `data` steps.
- SHOULD reference `@ERRORS.md` for error code definitions used in `error_paths`.
- SHOULD reference `@SCREENS.md` for visual representation of flow states.

## Validation Rules

1. `flows` MUST contain at least 1 entry.
2. Every `Flow.ux` MUST contain at least 1 step.
3. Every `Flow.data` MUST contain at least 1 step.
4. Every `UXStep.screen` SHOULD resolve to a page in `@UI.md` or a screen in `@SCREENS.md`.
5. Every `DataStep.service` SHOULD resolve to an endpoint in `@API.md` or an agent in `@RUNTIME.md`.
6. Every `ErrorPath.at` SHOULD reference a valid step within the same flow.

## Conflict Detection

- Flow `ux` steps MUST NOT reference screens that do not exist in `@UI.md` or `@SCREENS.md` when those files are present.
- Flow `data` steps MUST NOT reference API endpoints that do not exist in `@API.md` when that file is present.
- `error_paths` error conditions SHOULD align with error codes defined in `@ERRORS.md`.
- Circular flow dependencies (flow A triggers flow B triggers flow A) SHOULD be flagged as warnings.
