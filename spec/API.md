# API.md Specification

> Version: 0.1.0-draft | Status: Draft

## Purpose

API.md defines endpoint conventions, authentication, error handling, pagination, and versioning for the project's API surface.

## Frontmatter Schema

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | API name |
| `type` | `enum(rest\|graphql\|trpc\|grpc\|mixed)` | REQUIRED | API paradigm |
| `base_url` | `string` | OPTIONAL | Base URL (e.g., "/api/v1") |
| `auth` | `Auth` | REQUIRED | Authentication configuration |
| `versioning` | `Versioning` | OPTIONAL | API versioning strategy |
| `rate_limit` | `RateLimit` | OPTIONAL | Rate limiting rules |
| `error_format` | `ErrorFormat` | REQUIRED | Error response shape. MUST reference `@ERRORS.md`. |
| `pagination` | `Pagination` | OPTIONAL | Pagination strategy |
| `routers` | `map<string, Router>` | OPTIONAL | For tRPC or modular REST APIs |

### Auth

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `type` | `enum(bearer\|cookie\|api-key\|none)` | REQUIRED | Auth mechanism |
| `header` | `string` | OPTIONAL | Header name. Default: `"Authorization"` for bearer. |

### Versioning

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `strategy` | `enum(url\|header\|none)` | REQUIRED | How versions are distinguished |
| `current` | `string` | REQUIRED | Current version identifier |

### RateLimit

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `requests` | `number` | REQUIRED | Max requests per window |
| `window` | `string` | REQUIRED | Window duration (e.g., "1m", "1h") |

### ErrorFormat

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `type` | `string` | REQUIRED | Format name (e.g., "RFC 7807", "custom") |
| `fields` | `string[]` | REQUIRED | Fields in error response (e.g., ["code", "message", "details"]) |

### Pagination

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `type` | `enum(cursor\|offset\|none)` | REQUIRED | Pagination strategy |
| `default_limit` | `number` | OPTIONAL | Default page size |
| `max_limit` | `number` | OPTIONAL | Maximum page size |

### Router

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `description` | `string` | REQUIRED | Router purpose |
| `procedures` | `Procedure[]` | REQUIRED | Procedures in this router |

### Procedure

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Procedure name |
| `type` | `enum(query\|mutation)` | REQUIRED | Read or write operation |
| `auth` | `boolean` | REQUIRED | Whether authentication is required |
| `description` | `string` | REQUIRED | What this procedure does |

## Body Sections

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## Authentication` | REQUIRED | Auth flow description with token lifecycle. |
| `## Endpoints / Routers` | REQUIRED | Full endpoint or router listing with request/response shapes. |
| `## Error Handling` | REQUIRED | MUST reference `@ERRORS.md`. Describe how errors are returned. |
| `## Pagination` | OPTIONAL | Pagination examples. |
| `## Rate Limiting` | OPTIONAL | Rate limit behavior and headers. |
| `## Versioning` | OPTIONAL | Version migration guidance. |
| `## Examples` | SHOULD | At least 3 request/response examples. |

## Cross-References

- MUST reference `@ERRORS.md` for error code definitions.
- SHOULD reference `@SCHEMA.md` for data models behind endpoints.
- SHOULD reference `@SECURITY.md` for auth and authorization details.

## Validation Rules

1. Every `Procedure` MUST have a non-empty `description`.
2. `error_format.fields` MUST contain at least 1 field.
3. If `pagination` is defined, `max_limit` MUST be >= `default_limit`.

## Conflict Detection

- `auth.type` MUST match authentication methods declared in `@SECURITY.md`.
- Every model referenced in endpoint request/response MUST exist in `@SCHEMA.md`.
- Error codes used in `## Examples` MUST be defined in `@ERRORS.md`.
