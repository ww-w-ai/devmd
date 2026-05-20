# ERRORS.md Specification

> Version: 0.1.0-draft | Status: Draft

## Purpose

ERRORS.md defines the error code registry, exception hierarchy, retry policies, and error propagation rules.

## Frontmatter Schema

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Error system name |
| `codes` | `map<string, ErrorCode>` | REQUIRED | Keyed by error code string (e.g., "AUTH_INVALID_TOKEN") |
| `exception_hierarchy` | `ExceptionClass[]` | OPTIONAL | Exception class tree |
| `retry_policy` | `RetryPolicy` | OPTIONAL | Default retry behavior |

### ErrorCode

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `http_status` | `number` | REQUIRED | HTTP status code. MUST be 100-599. |
| `message` | `string` | REQUIRED | Internal/developer-facing message |
| `user_message` | `string` | OPTIONAL | User-facing message. SHOULD follow `@BRAND.md` tone. |
| `retryable` | `boolean` | OPTIONAL | Default: `false`. Whether client should retry. |
| `recovery` | `string` | OPTIONAL | Suggested recovery action |

### ExceptionClass

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `class` | `string` | REQUIRED | Exception class name |
| `extends` | `string` | OPTIONAL | Parent class. MUST reference another class in this list or a language built-in. |
| `description` | `string` | REQUIRED | When this exception is thrown |

### RetryPolicy

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `max_retries` | `number` | REQUIRED | Maximum retry attempts |
| `backoff` | `enum(linear\|exponential\|fixed)` | REQUIRED | Backoff strategy |
| `base_delay` | `string` | REQUIRED | Initial delay (e.g., "100ms", "1s") |

## Body Sections

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## Error Codes` | REQUIRED | Full error code table with examples. |
| `## Exception Hierarchy` | OPTIONAL | Class diagram or tree showing inheritance. MAY use Mermaid. |
| `## Retry Policy` | OPTIONAL | When and how to retry. Include jitter guidance. |
| `## Error Propagation` | OPTIONAL | How errors flow between layers (domain -> application -> API). |
| `## User-Facing Messages` | OPTIONAL | Tone and copy rules for user messages. SHOULD reference `@BRAND.md`. |

## Cross-References

- MUST be referenced by `@API.md` for error response format.
- SHOULD reference `@BRAND.md` for user-facing message tone.
- SHOULD reference `@LOGGING.md` for error logging behavior.

## Validation Rules

1. Every `http_status` MUST be an integer between 100 and 599 inclusive.
2. Error code keys MUST be UPPER_SNAKE_CASE.
3. Every error code MUST have a non-empty `message`.
4. `ExceptionClass.extends` MUST reference a `class` defined in the same `exception_hierarchy` array or be a recognized language built-in (e.g., "Error", "Exception").

## Conflict Detection

- Every error code referenced in `@API.md` examples or error handling sections MUST be defined in `codes`.
- `user_message` tone MUST NOT contradict `@BRAND.md#voice` if both exist.
- `retryable: true` codes SHOULD be logged at a level that matches `@LOGGING.md` retry event configuration.
