# LOGGING.md Specification

> Version: 0.1.0-draft | Status: Draft

## Purpose

LOGGING.md defines log format, levels, request tracing, audit events, PII masking, and retention policies.

## Frontmatter Schema

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Logging system name |
| `framework` | `string` | REQUIRED | Logging library (e.g., "pino", "winston", "structlog", "slog") |
| `format` | `enum(json\|text\|structured)` | REQUIRED | Output format |
| `levels` | `map<string, LogLevel>` | REQUIRED | Log level definitions |
| `request_id` | `RequestId` | OPTIONAL | Request tracing configuration |
| `pii_masking` | `PIIMasking` | OPTIONAL | PII protection rules |
| `audit_events` | `map<string, AuditEvent>` | OPTIONAL | Keyed by event name |

### LogLevel

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `priority` | `number` | REQUIRED | Numeric priority. Lower = more severe. |
| `description` | `string` | REQUIRED | When to use this level |

### RequestId

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `header` | `string` | REQUIRED | HTTP header carrying the request ID (e.g., "X-Request-Id") |
| `generator` | `string` | REQUIRED | ID generation strategy (e.g., "uuid-v4", "ulid", "nanoid") |

### PIIMasking

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `enabled` | `boolean` | REQUIRED | Whether PII masking is active |
| `patterns` | `string[]` | REQUIRED | Field name patterns to mask (e.g., "email", "password", "ssn") |

### AuditEvent

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `description` | `string` | REQUIRED | What triggers this event |
| `severity` | `string` | REQUIRED | Log level name. MUST reference a key in `levels`. |
| `fields` | `string[]` | REQUIRED | Fields captured in the audit log entry |

## Body Sections

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## Log Format` | REQUIRED | Example log output. MUST show at least one structured log entry. |
| `## Log Levels` | REQUIRED | When to use each level. Include examples of correct level selection. |
| `## Request Tracing` | OPTIONAL | How request IDs propagate across services. |
| `## Audit Events` | OPTIONAL | Expanded event descriptions with sample payloads. |
| `## PII Masking` | OPTIONAL | Masking strategy details and field pattern documentation. |
| `## Retention` | OPTIONAL | Log storage duration, rotation, and archival policies. |

## Cross-References

- SHOULD reference `@ERRORS.md` for error-level logging conventions.
- SHOULD reference `@SECURITY.md` for security audit event alignment.
- SHOULD reference `@RUNTIME.md` for agent/job logging patterns.

## Validation Rules

1. `levels` MUST contain at least 2 entries.
2. `LogLevel.priority` values MUST be unique across all levels.
3. `AuditEvent.severity` MUST reference a key defined in `levels`.
4. `PIIMasking.patterns` MUST contain at least 1 entry when `enabled` is `true`.

## Conflict Detection

- Audit events SHOULD cover all authentication events defined in `@SECURITY.md`.
- If `@ERRORS.md` defines retryable errors, retry attempts SHOULD have a corresponding log level defined here.
- `format` SHOULD be consistent with observability tooling declared in `@INFRA.md` if present.
