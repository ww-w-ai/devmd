# RUNTIME.md Specification

> Version: 0.1.0-draft | Status: Draft | DevMD Unique Proposal

## Purpose

RUNTIME.md specifies job and process execution infrastructure — queues, cron schedules, webhooks, and email. It complements `@HARNESS.md` (agent control plane) and `@INFRA.md` (infrastructure intent) by defining the execution layer for background work.

## Frontmatter Schema

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Runtime configuration name |
| `job_provider` | `JobProvider` | OPTIONAL | Background job framework |
| `queues` | `map<string, Queue>` | OPTIONAL | Named job queues |
| `cron` | `CronJob[]` | OPTIONAL | Scheduled tasks |
| `webhooks` | `Webhooks` | OPTIONAL | Inbound and outbound webhook config |
| `email` | `Email` | OPTIONAL | Email sending configuration |

### JobProvider

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `type` | `enum(bullmq\|celery\|arq\|sidekiq\|inngest\|custom)` | REQUIRED | Job framework |
| `config` | `map<string, any>` | OPTIONAL | Provider-specific configuration |

### Queue

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `description` | `string` | REQUIRED | What jobs this queue processes |
| `concurrency` | `number` | OPTIONAL | Maximum concurrent workers |
| `timeout` | `string` | OPTIONAL | Per-job timeout (e.g., `30s`, `5m`) |
| `retry` | `RetryPolicy` | OPTIONAL | Retry behavior |

### RetryPolicy

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `max` | `number` | REQUIRED | Maximum retry attempts |
| `backoff` | `string` | OPTIONAL | Backoff strategy (e.g., `exponential`, `linear`, `fixed`) |
| `dead_letter` | `boolean` | OPTIONAL | Whether failed jobs move to a dead-letter queue |

### CronJob

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Job identifier |
| `schedule` | `string` | REQUIRED | Cron expression (5- or 6-field) |
| `handler` | `string` | REQUIRED | Function or module path |
| `description` | `string` | REQUIRED | What this job does |

### Webhooks

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `outbound` | `OutboundWebhook[]` | OPTIONAL | Events sent to external systems |
| `inbound` | `InboundWebhook[]` | OPTIONAL | Events received from external systems |

**OutboundWebhook**: `{event: string, payload_schema: string, retry: RetryPolicy}` — `event` and `payload_schema` REQUIRED.
**InboundWebhook**: `{path: string, validation: string, handler: string}` — all REQUIRED.

### Email

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `provider` | `string` | REQUIRED | Email service (e.g., `ses`, `sendgrid`, `resend`, `postmark`) |
| `templates` | `string[]` | OPTIONAL | Template identifiers |

## Body Sections

Sections MUST appear in this order when present.

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## Job Queues` | OPTIONAL | Queue topology, priority rules, worker configuration. |
| `## Cron Jobs` | OPTIONAL | Schedule rationale, timezone considerations. |
| `## Webhooks` | OPTIONAL | Payload examples, signature verification. |
| `## Email` | OPTIONAL | Template descriptions, sending rules. |
| `## Retry Policy` | OPTIONAL | Global retry defaults and dead-letter handling. |

## Cross-References

- SHOULD reference `@INFRA.md` for queue and email provider infrastructure.
- SHOULD reference `@ERRORS.md` for job error classification and retry behavior.
- SHOULD reference `@CONFIG.md` for provider configuration and credentials.

## Validation Rules

1. Every `CronJob.schedule` MUST be a valid cron expression.
2. `CronJob.name` MUST be unique within the `cron` array.
3. `Queue` names MUST be unique within the `queues` map.
4. `RetryPolicy.max` MUST be a positive integer.
5. `InboundWebhook.path` MUST start with `/`.

## Conflict Detection

- `job_provider.type` SHOULD be consistent with queue infrastructure in `@INFRA.md#queue`.
- `email.provider` SHOULD match the email provider in `@INFRA.md` if declared.
- Outbound webhook events SHOULD correspond to domain events in `@SCHEMA.md` or `@API.md`.
