---
devmd: logging
version: "1.0"
project: documenso
updated: 2026-05-13

library: pino
format: json
transport: stdout

log_levels:
  - level: fatal
    code: 60
    usage: "Process is about to crash. Immediate action required."
  - level: error
    code: 50
    usage: "Operation failed. AppError or unexpected exception."
  - level: warn
    code: 40
    usage: "Recoverable issue. Deprecation warnings, rate limit approaches."
  - level: info
    code: 30
    usage: "Normal operations. Request lifecycle, job completion, document status changes."
  - level: debug
    code: 20
    usage: "Diagnostic info. SQL queries (dev only), tRPC procedure calls."
  - level: trace
    code: 10
    usage: "Extremely verbose. Rarely used in production."

default_level:
  production: info
  development: debug
  test: warn

observability:
  analytics: PostHog
  profiling: Datadog
  telemetry: built-in
---

# LOGGING.md — Documenso

## Structured Log Format

All logs are JSON objects written to stdout via Pino:

```json
{
  "level": 30,
  "time": 1715600000000,
  "pid": 1,
  "hostname": "documenso-web-abc123",
  "msg": "Document sent successfully",
  "requestId": "req_a1b2c3d4",
  "userId": 42,
  "envelopeId": 123,
  "recipientCount": 3,
  "duration": 245
}
```

### Standard Fields

| Field | Type | Always Present | Description |
|---|---|---|---|
| `level` | number | Yes | Pino log level (10-60) |
| `time` | number | Yes | Unix timestamp ms |
| `pid` | number | Yes | Process ID |
| `hostname` | string | Yes | Container hostname |
| `msg` | string | Yes | Human-readable message |
| `requestId` | string | Per-request | Unique request identifier for correlation |
| `userId` | number | When authenticated | Acting user ID |
| `teamId` | number | When scoped | Team context |
| `envelopeId` | number | When relevant | Document/template envelope ID |
| `duration` | number | For timed ops | Operation duration in ms |
| `error` | object | On errors | Serialized error with stack (dev only) |

## Request Logging

HTTP requests are logged at the Hono middleware level:

```json
{
  "level": 30,
  "msg": "request completed",
  "requestId": "req_a1b2c3d4",
  "method": "POST",
  "path": "/trpc/document.sendDocument",
  "statusCode": 200,
  "duration": 245,
  "userAgent": "Mozilla/5.0 ...",
  "ip": "203.0.113.1"
}
```

### Request ID Propagation

Every incoming request gets a unique `requestId` (generated or extracted from `X-Request-Id` header). This ID:

1. Attaches to the Pino child logger for the request
2. Passes through to tRPC context
3. Includes in background job payloads for tracing
4. Returns in response headers as `X-Request-Id`

## Error Logging

### AppError (Expected)

```json
{
  "level": 40,
  "msg": "Document not found",
  "requestId": "req_a1b2c3d4",
  "error": {
    "code": "NOT_FOUND",
    "httpStatus": 404,
    "userMessage": "Document not found"
  },
  "userId": 42,
  "envelopeId": 999
}
```

AppErrors at `warn` level (4xx) or `error` level (5xx).

### Unexpected Error

```json
{
  "level": 50,
  "msg": "Unexpected error in document.sendDocument",
  "requestId": "req_a1b2c3d4",
  "error": {
    "type": "TypeError",
    "message": "Cannot read properties of undefined",
    "stack": "TypeError: Cannot read properties... (dev only)"
  }
}
```

Unexpected errors at `error` level. Stack traces included in development, omitted in production.

## Audit Events

Document lifecycle events are logged both to the database (`DocumentAuditLog`) and to structured logs. See @SCHEMA.md#documentauditlog for the 18 event types.

```json
{
  "level": 30,
  "msg": "audit: DOCUMENT_SENT",
  "envelopeId": 123,
  "userId": 42,
  "recipientCount": 3,
  "auditLogId": "cla1b2c3d4"
}
```

### Audit Event Types

| Event | Trigger |
|---|---|
| `DOCUMENT_CREATED` | New envelope created |
| `DOCUMENT_OPENED` | Recipient opens signing view |
| `DOCUMENT_SENT` | Document transitioned to PENDING |
| `DOCUMENT_COMPLETED` | All recipients done, document sealed |
| `DOCUMENT_DELETED` | Document soft-deleted |
| `DOCUMENT_FIELD_INSERTED` | Recipient fills a field |
| `DOCUMENT_FIELD_UNINSERTED` | Recipient clears a field |
| `DOCUMENT_META_UPDATED` | Document metadata changed |
| `DOCUMENT_RECIPIENT_COMPLETED` | Individual recipient finished |
| `DOCUMENT_RECIPIENT_REJECTED` | Recipient rejected document |
| `DOCUMENT_RECIPIENT_ADDED` | Recipient added to envelope |
| `DOCUMENT_RECIPIENT_REMOVED` | Recipient removed from envelope |
| `EMAIL_SENT` | Email notification dispatched |
| `DOCUMENT_MOVED_TO_TEAM` | Document transferred between teams |
| `DOCUMENT_GLOBAL_AUTH_*` | Auth settings changed (2 events) |
| `DOCUMENT_RECIPIENT_AUTH_*` | Recipient auth settings changed (2 events) |

## PII Masking

### Logged (Safe)

- User ID (numeric)
- Envelope ID (numeric)
- Team ID (numeric)
- Request path and method
- IP address (for audit trail — required for legal compliance)
- Email addresses in audit logs only (legal requirement for signing)

### Never Logged

- Passwords or password hashes
- Session tokens or API token values
- PDF document content or binary data
- Signature image data
- OAuth tokens (access/refresh)
- 2FA secrets
- Webhook signing secrets

### Redacted in Non-Audit Contexts

- Full email addresses (show `u***@domain.com` pattern)
- Recipient names (show first initial only)

## Background Job Logging

Background jobs log their lifecycle:

```json
{"level":30,"msg":"job:started","jobId":"cla1b2c3","jobName":"send-signing-email","attempt":1}
{"level":30,"msg":"job:completed","jobId":"cla1b2c3","jobName":"send-signing-email","duration":1234}
{"level":50,"msg":"job:failed","jobId":"cla1b2c3","jobName":"send-signing-email","attempt":2,"error":{...}}
{"level":40,"msg":"job:max-retries","jobId":"cla1b2c3","jobName":"send-signing-email","attempts":3}
```

## Observability Stack

| Tool | Purpose | Environment |
|---|---|---|
| **Pino** | Structured JSON logging | All |
| **PostHog** | Product analytics, feature flags | Production |
| **Datadog** | APM, profiling, infrastructure monitoring | Production (cloud) |
| **Built-in telemetry** | Self-host usage metrics (opt-in) | Self-hosted |

### PostHog Events

Product analytics events tracked via PostHog (separate from structured logs):

- Document created/sent/completed
- Template created/used
- User signed up/logged in
- Feature usage (direct links, API tokens, webhooks)

## Development Logging

In development, Pino is configured with `pino-pretty` for human-readable output:

```
[12:34:56.789] INFO: Document sent successfully
    requestId: "req_a1b2c3d4"
    userId: 42
    envelopeId: 123
    recipientCount: 3
    duration: 245ms
```

## Cross-References

- Error codes and hierarchy: @ERRORS.md
- Audit log database model: @SCHEMA.md#documentauditlog
- Background job lifecycle: @RUNTIME.md#background-jobs
- Security context for audit: @SECURITY.md#audit-trail
