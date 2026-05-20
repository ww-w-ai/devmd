---
devmd: runtime
version: "1.0"
project: Twenty CRM

entry_points:
  - name: api
    command: "node dist/src/main api"
    port: 3000
    protocol: "HTTP + WebSocket (SSE)"
    description: "GraphQL API server + REST + SSE real-time"
  - name: worker
    command: "node dist/src/main worker"
    description: "BullMQ job processor — async tasks, email sync, webhooks"
  - name: cli
    command: "node dist/src/main cli"
    description: "Administrative CLI — migrations, seed, maintenance"

worker:
  queue_system: BullMQ
  broker: Redis
  concurrency: "configurable (default: 10 per worker instance)"
  queues:
    - name: email-sync
      description: "Sync emails from connected IMAP accounts"
      schedule: "Every 5 minutes per connected account"
      retry: "3 attempts, exponential backoff"
      timeout: "5 minutes"
    - name: calendar-sync
      description: "Sync calendar events from CalDAV accounts"
      schedule: "Every 5 minutes per connected account"
      retry: "3 attempts, exponential backoff"
      timeout: "5 minutes"
    - name: webhook-dispatch
      description: "Send outbound webhook HTTP calls"
      retry: "3 attempts (1s, 5s, 30s backoff)"
      timeout: "30 seconds"
      dead_letter: true
    - name: workflow-execution
      description: "Execute workflow actions (send email, create record, etc.)"
      retry: "2 attempts"
      timeout: "2 minutes"
    - name: data-enrichment
      description: "Enrich company/person data from external sources"
      retry: "2 attempts"
      timeout: "1 minute"
    - name: search-indexing
      description: "Index/re-index records for full-text search"
      retry: "3 attempts"
      timeout: "30 seconds"
    - name: export
      description: "Generate CSV/Excel export files"
      retry: "1 attempt"
      timeout: "10 minutes"
    - name: import
      description: "Process CSV import files"
      retry: "1 attempt"
      timeout: "30 minutes"
    - name: cleanup
      description: "Periodic cleanup (orphan files, expired tokens, stale locks)"
      schedule: "Daily at 2 AM UTC"
      retry: "1 attempt"
      timeout: "30 minutes"

cron_jobs:
  - name: email-sync-scheduler
    schedule: "*/5 * * * *"
    description: "Enqueue email sync jobs for all active connected accounts"
  - name: calendar-sync-scheduler
    schedule: "*/5 * * * *"
    description: "Enqueue calendar sync jobs for all active connected accounts"
  - name: token-cleanup
    schedule: "0 */6 * * *"
    description: "Remove expired refresh tokens and app tokens"
  - name: workspace-cleanup
    schedule: "0 3 * * *"
    description: "Drop schemas for workspaces deleted > 30 days ago"
  - name: analytics-aggregation
    schedule: "0 * * * *"
    description: "Aggregate hourly metrics to ClickHouse materialized views"
  - name: billing-sync
    schedule: "0 0 * * *"
    description: "Sync billing subscription status, check entitlements"

real_time:
  protocol: SSE (Server-Sent Events)
  endpoint: "/api/events"
  auth: "JWT in query parameter or cookie"
  events:
    - name: record.created
      payload: "{ objectName, recordId, data }"
    - name: record.updated
      payload: "{ objectName, recordId, changedFields, data }"
    - name: record.deleted
      payload: "{ objectName, recordId }"
    - name: view.updated
      payload: "{ viewId, changes }"
    - name: notification
      payload: "{ type, message, recordId }"
    - name: schema.changed
      payload: "{ schemaVersion }"
  scoping: "Events scoped to workspace. User receives events for their workspace only."

triggers:
  database_mutation:
    description: "twenty-orm emits events on all create/update/delete operations"
    consumers:
      - workflow-engine (evaluate trigger conditions)
      - webhook-dispatcher (match registered webhooks)
      - sse-broadcaster (push to connected clients)
      - search-indexer (update search index)
      - audit-logger (record change in audit trail)
---

# Twenty CRM — Runtime Architecture

## Three Entry Points

The twenty-server package runs as three separate processes. See @ARCHITECTURE.md.

```
┌──────────────────────────────────────────────────────┐
│                  twenty-server                        │
│                                                      │
│  ┌────────────┐  ┌────────────┐  ┌────────────────┐  │
│  │ API Server │  │   Worker   │  │     CLI        │  │
│  │ (main api) │  │(main worker│  │ (main cli)     │  │
│  │            │  │            │  │                │  │
│  │ GraphQL    │  │ BullMQ     │  │ Migrations     │  │
│  │ REST       │  │ job        │  │ Seed           │  │
│  │ SSE        │  │ processor  │  │ Maintenance    │  │
│  │ WebSocket  │  │            │  │                │  │
│  └─────┬──────┘  └─────┬──────┘  └────────────────┘  │
│        │               │                             │
│        └───────┬───────┘                             │
│                │                                      │
│         ┌──────▼──────┐                              │
│         │  Shared     │                              │
│         │  Engine     │                              │
│         │  Domain     │                              │
│         └─────────────┘                              │
└──────────────────────────────────────────────────────┘
```

## BullMQ Worker

The worker process consumes jobs from Redis-backed BullMQ queues. Each queue has dedicated concurrency, timeout, and retry settings.

### Job Lifecycle

```
Producer (API Server or Cron)
  │  Enqueue job with payload + options
  ▼
Redis Queue
  │  FIFO ordering, priority support
  ▼
Worker Process
  │  Dequeue job
  │  Create OpenTelemetry span (trace propagated from producer)
  │  Execute job handler
  │  On success: mark completed, emit completion event
  │  On failure: retry (if attempts remaining) or move to dead letter
  ▼
Result
  │  Success → log + metrics
  │  Failure → log + metrics + Sentry alert (if max retries exhausted)
```

### Queue Configuration

```typescript
// email-sync queue
new Queue('email-sync', {
  connection: redisConnection,
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 5000 },
    timeout: 300000, // 5 minutes
    removeOnComplete: { age: 86400 }, // keep completed jobs for 24h
    removeOnFail: { age: 604800 },    // keep failed jobs for 7 days
  },
});

// webhook-dispatch queue
new Queue('webhook-dispatch', {
  connection: redisConnection,
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'custom', delay: [1000, 5000, 30000] },
    timeout: 30000, // 30 seconds
  },
});
```

## Email & Calendar Sync

### Email Sync (IMAP)

```
Cron (every 5 min)
  │  Finds all active connected accounts
  │  Enqueues email-sync job per account
  ▼
Worker: email-sync
  │
  ├── Connect to IMAP server (Gmail, Outlook, generic)
  ├── Fetch new messages since last sync cursor
  ├── For each message:
  │     ├── Parse headers, body, attachments
  │     ├── Match participants to existing Person records
  │     ├── Create/update Message + MessageParticipant records
  │     ├── Link to MessageThread (by In-Reply-To / References headers)
  │     └── Emit message.created event
  ├── Update sync cursor on ConnectedAccount
  └── Handle errors:
        ├── Auth failure → mark authFailedAt, notify user
        ├── Rate limit → retry with backoff
        └── Network error → retry on next cycle
```

### Calendar Sync (CalDAV)

```
Cron (every 5 min)
  │  Enqueues calendar-sync job per account
  ▼
Worker: calendar-sync
  │
  ├── Connect to CalDAV server
  ├── Fetch changed events since last sync token
  ├── For each event:
  │     ├── Parse iCal data (VEVENT)
  │     ├── Match attendees to existing Person records
  │     ├── Create/update CalendarEvent + CalendarEventParticipant
  │     └── Emit calendarEvent.created/updated event
  └── Update sync token on ConnectedAccount
```

## Webhook Dispatch

Triggered by database mutation events. See @API.md#webhooks.

```
Database Mutation (twenty-orm event)
  │
  ▼
WebhookDispatcher (event listener)
  │  Match event against registered webhooks
  │  For each matching webhook:
  │    Enqueue webhook-dispatch job
  ▼
Worker: webhook-dispatch
  │
  ├── Build payload (event type, record data, workspace ID)
  ├── POST to webhook target URL
  │     Headers: Content-Type: application/json, X-Twenty-Signature: HMAC-SHA256
  ├── On 2xx: success, mark delivered
  ├── On 4xx: log warning, no retry (client error)
  ├── On 5xx: retry with backoff (1s, 5s, 30s)
  └── On max retries exhausted: move to dead letter queue, alert
```

### Webhook Signature Verification

```
Signature = HMAC-SHA256(webhook_secret, request_body)
Header: X-Twenty-Signature: sha256={signature}
```

Consumers verify by computing HMAC with their webhook secret and comparing.

## Real-Time Events (SSE)

Server-Sent Events stream for live UI updates.

```
Client connects: GET /api/events
  Headers: Authorization: Bearer {token}
  │
  ▼
Server validates JWT, extracts workspaceId
  │
  ▼
Subscribe to workspace event bus (Redis pub/sub)
  │
  ▼
On database mutation → event published to Redis channel
  │
  ▼
SSE endpoint streams event to connected clients:
  data: {"event":"record.updated","objectName":"person","recordId":"uuid","changedFields":["jobTitle"]}

Client (Apollo Client) receives event:
  │
  ├── record.updated → invalidate/update Apollo cache for that record
  ├── record.created → add to list queries if matches current filters
  ├── record.deleted → remove from Apollo cache
  ├── schema.changed → refetch GraphQL schema
  └── notification → show toast notification
```

## Workflow Engine

Automation workflows defined by users. See @GLOSSARY.md#workflow.

### Trigger Types

| Trigger | Description |
|---|---|
| `record.created` | When a record of specified type is created |
| `record.updated` | When specified fields on a record change |
| `record.deleted` | When a record is soft-deleted |
| `scheduled` | Cron-based trigger (e.g., every Monday 9 AM) |
| `manual` | User clicks "Run workflow" button |

### Action Types

| Action | Description |
|---|---|
| `create-record` | Create a new record of any type |
| `update-record` | Update fields on the triggering record or a related record |
| `delete-record` | Soft-delete a record |
| `send-email` | Send email via connected account |
| `create-task` | Create a follow-up task |
| `call-webhook` | HTTP request to external URL |
| `wait` | Delay execution by duration |
| `condition` | Branch based on field values |

### Execution Flow

```
Trigger event occurs
  │
  ▼
WorkflowEngine evaluates all active workflows for this trigger type
  │  Filter by conditions (e.g., only when stage = "CLOSED_WON")
  ▼
Enqueue workflow-execution job per matching workflow
  │
  ▼
Worker: workflow-execution
  │  Execute actions in sequence
  │  For each action:
  │    ├── Resolve dynamic values ({{record.name}}, {{user.email}})
  │    ├── Execute action (create record, send email, etc.)
  │    ├── Log execution step
  │    └── On failure: retry or skip based on configuration
  │
  ▼
Update workflow execution status (success/failed/partial)
Log to audit trail
```

## Cron Job Schedule Summary

| Job | Schedule | Duration |
|---|---|---|
| Email sync scheduler | Every 5 min | < 10s |
| Calendar sync scheduler | Every 5 min | < 10s |
| Token cleanup | Every 6 hours | < 1 min |
| Workspace cleanup | Daily 3 AM | < 5 min |
| Analytics aggregation | Hourly | < 2 min |
| Billing sync | Daily midnight | < 1 min |

## Cross-References

- @ARCHITECTURE.md — Server entry points and engine layer
- @SCHEMA.md — Objects involved in email/calendar sync
- @API.md#webhooks — Webhook configuration and payload format
- @ERRORS.md — Worker job error handling and retry policies
- @LOGGING.md — Worker job tracing and metrics
- @INFRA.md — Worker scaling and queue monitoring
- @SECURITY.md — Webhook signature verification
