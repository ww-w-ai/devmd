---
devmd: runtime
version: "1.0"
project: documenso
updated: 2026-05-13

background_jobs:
  providers: [local, bullmq, inngest]
  default: local
  model: BackgroundJob
  status_enum: [PENDING, PROCESSING, COMPLETED, FAILED]
  max_retries: 3

cron_jobs:
  - name: seal-documents
    schedule: "*/5 * * * *"
    description: "Check for documents ready to seal (all recipients completed)"
  - name: send-reminder-emails
    schedule: "0 9 * * *"
    description: "Send reminder emails for unsigned documents"
  - name: expire-documents
    schedule: "0 * * * *"
    description: "Expire documents past their deadline"
  - name: cleanup-expired-sessions
    schedule: "0 2 * * *"
    description: "Remove expired sessions from database"
  - name: process-webhooks
    schedule: "*/1 * * * *"
    description: "Process pending webhook deliveries"

job_types:
  - name: send-signing-email
    trigger: "Document sent (DRAFT → PENDING)"
    payload: "{envelopeId, recipientId}"
    priority: high
  - name: send-completion-email
    trigger: "Document completed (all signed + sealed)"
    payload: "{envelopeId}"
    priority: high
  - name: seal-document
    trigger: "All recipients completed"
    payload: "{envelopeId}"
    priority: critical
  - name: send-reminder-email
    trigger: "Cron job (daily)"
    payload: "{envelopeId, recipientId}"
    priority: normal
  - name: dispatch-webhook
    trigger: "Document event occurs"
    payload: "{webhookId, eventType, data}"
    priority: normal
  - name: generate-audit-pdf
    trigger: "User requests audit log download"
    payload: "{envelopeId}"
    priority: low
---

# RUNTIME.md — Documenso

## Background Job System

Documenso uses a **swappable background job system** with three provider options. All providers implement the same interface, selected via `NEXT_PRIVATE_JOBS_TRANSPORT` environment variable.

### Architecture

```
Request Handler
    │
    ▼
enqueueJob({ name, payload, priority })
    │
    ├── local: INSERT INTO BackgroundJob
    ├── bullmq: bull.add(name, payload)
    └── inngest: inngest.send(event)
    
    ▼ (async processing)
    
Job Worker
    │
    ▼
processJob(job)
    │
    ├── Execute domain logic from @documenso/lib
    ├── Update job status (PROCESSING → COMPLETED | FAILED)
    └── Retry on failure (up to maxRetries)
```

### Provider Details

#### Local (Database Queue)

The default provider for simple self-hosting. Uses the `BackgroundJob` Prisma model (see @SCHEMA.md#backgroundjob) as a job queue.

```
Workflow:
1. Job enqueued → INSERT row with status=PENDING
2. Polling worker runs every 5 seconds
3. Worker claims job → UPDATE status=PROCESSING
4. Worker executes job handler
5. Success → UPDATE status=COMPLETED, set completedAt
6. Failure → INCREMENT retryCount, UPDATE status=PENDING (for retry) or FAILED (max retries)
```

**Trade-offs**:
- No external dependencies (just PostgreSQL)
- Single-instance only (no distributed processing)
- Polling-based (5s latency between job creation and pickup)
- Adequate for small to medium deployments

#### BullMQ (Redis Queue)

Production-grade job queue backed by Redis.

```
Workflow:
1. Job enqueued → Redis LPUSH
2. Worker BRPOP (blocking, near-zero latency)
3. Job processing with Redis-backed locking
4. Built-in retry with configurable backoff
5. Dead letter queue for permanently failed jobs
```

**Trade-offs**:
- Requires Redis instance
- Multi-instance safe (distributed workers)
- Sub-second job pickup
- Built-in job scheduling (delayed jobs, repeatable jobs)
- Dashboard available via BullBoard

**Redis connection**: `NEXT_PRIVATE_REDIS_URL`

#### Inngest (Event-Driven Functions)

Serverless-friendly option using Inngest's event-driven architecture.

```
Workflow:
1. Job enqueued → inngest.send({ name, data })
2. Inngest routes event to registered function
3. Function executes with built-in retry, concurrency control
4. Inngest manages state, retries, and observability
```

**Trade-offs**:
- External service dependency
- Best for serverless deployments (Vercel, Railway)
- Built-in observability dashboard
- Automatic retry with backoff
- Event replay capability

## Job Types

### Critical Priority

#### seal-document

The most important background job. Cryptographically seals the PDF after all recipients complete.

```typescript
{
  name: 'seal-document',
  payload: { envelopeId: number },
  priority: 'critical',
  maxRetries: 5,  // Higher retry count for critical job
  timeout: 120_000  // 2 minutes (PDF processing can be slow)
}

// Execution steps:
// 1. Load envelope with all recipients, fields, signatures
// 2. Render signatures onto PDF pages (image overlay)
// 3. Generate document hash
// 4. Sign with certificate (local P12 or HSM) — see @SECURITY.md#pdf-cryptographic-signing
// 5. Store sealed PDF via storage provider
// 6. Update envelope status → COMPLETED
// 7. Enqueue completion notification emails
// 8. Write audit log entry
```

### High Priority

#### send-signing-email

Sends email notification to each recipient when a document is sent.

```typescript
{
  name: 'send-signing-email',
  payload: { envelopeId: number, recipientId: number },
  priority: 'high',
  maxRetries: 3,
  timeout: 30_000
}

// Execution:
// 1. Load recipient and envelope details
// 2. Render email template (React Email)
// 3. Send via configured email provider — see @INFRA.md#email-provider-interface
// 4. Update recipient sendStatus → SENT
// 5. Write audit log entry (EMAIL_SENT)
```

#### send-completion-email

Notifies all recipients when document signing is complete.

```typescript
{
  name: 'send-completion-email',
  payload: { envelopeId: number },
  priority: 'high',
  maxRetries: 3,
  timeout: 30_000
}
```

### Normal Priority

#### dispatch-webhook

Delivers webhook events to registered URLs.

```typescript
{
  name: 'dispatch-webhook',
  payload: {
    webhookId: string,
    eventType: string,
    data: Record<string, unknown>
  },
  priority: 'normal',
  maxRetries: 3,
  timeout: 30_000
}

// Execution:
// 1. Load webhook config (URL, secret, enabled)
// 2. Build payload with event data
// 3. Sign payload with HMAC-SHA256 (if secret configured)
// 4. POST to webhook URL with X-Webhook-Signature header
// 5. Verify 2xx response
// 6. On failure: retry with exponential backoff (1m, 5m, 30m)
```

#### send-reminder-email

Daily reminder to recipients who haven't signed.

```typescript
{
  name: 'send-reminder-email',
  payload: { envelopeId: number, recipientId: number },
  priority: 'normal',
  maxRetries: 3,
  timeout: 30_000
}
```

### Low Priority

#### generate-audit-pdf

Generates a formatted PDF of the document's audit trail for download.

```typescript
{
  name: 'generate-audit-pdf',
  payload: { envelopeId: number },
  priority: 'low',
  maxRetries: 2,
  timeout: 60_000
}
```

## Cron Jobs

Scheduled tasks run via the job provider's scheduling capability:

| Job | Schedule | Description |
|---|---|---|
| `seal-documents` | Every 5 min | Scan for documents where all recipients completed but not yet sealed. Catches any missed seal triggers. |
| `send-reminder-emails` | Daily 9:00 UTC | Send reminders to recipients of PENDING documents past reminder threshold. |
| `expire-documents` | Hourly | Transition documents past expiry date to expired state. |
| `cleanup-expired-sessions` | Daily 2:00 UTC | Delete expired session records from database. |
| `process-webhooks` | Every 1 min | Retry failed webhook deliveries. Pick up any webhooks that weren't processed inline. |

### Cron Implementation by Provider

- **Local**: Node.js `setInterval` or `node-cron` running in the same process
- **BullMQ**: `bull.add(name, payload, { repeat: { cron: '...' } })`
- **Inngest**: `createFunction({ id, cron: '...' }, handler)`

## Retry Policy

| Priority | Max Retries | Backoff Strategy | Timeout |
|---|---|---|---|
| Critical | 5 | Exponential (30s, 1m, 5m, 15m, 30m) | 120s |
| High | 3 | Exponential (10s, 1m, 5m) | 30s |
| Normal | 3 | Exponential (1m, 5m, 30m) | 30s |
| Low | 2 | Exponential (5m, 30m) | 60s |

### Failure Handling

```
Job fails:
  retryCount < maxRetries?
    YES → Increment retryCount, set status=PENDING, schedule retry after backoff
    NO  → Set status=FAILED, log error, alert if critical
```

Failed critical jobs (seal-document) trigger alerts via logging. See @LOGGING.md#background-job-logging.

## Webhook Events

Webhook payloads are dispatched for these document events:

| Event | Trigger |
|---|---|
| `DOCUMENT_SENT` | Document transitions from DRAFT to PENDING |
| `DOCUMENT_COMPLETED` | Document is sealed and all recipients done |
| `DOCUMENT_REJECTED` | A recipient rejects the document |
| `RECIPIENT_COMPLETED` | Individual recipient completes signing |
| `RECIPIENT_REJECTED` | Individual recipient rejects |

### Webhook Payload Format

```json
{
  "event": "DOCUMENT_COMPLETED",
  "timestamp": "2026-05-13T12:00:00Z",
  "data": {
    "envelopeId": 123,
    "documentTitle": "Service Agreement",
    "status": "COMPLETED",
    "recipients": [
      { "email": "signer@example.com", "role": "SIGNER", "status": "SIGNED" }
    ]
  }
}
```

### Webhook Security

```
X-Webhook-Signature: sha256=<HMAC-SHA256 of body with webhook secret>
X-Webhook-Timestamp: 1715600000
```

Receivers should validate signature and reject requests older than 5 minutes (replay protection).

## Email Sending

Email templates are React components rendered by `@documenso/email`:

| Template | Trigger |
|---|---|
| `signing-invitation` | Document sent to recipient |
| `signing-reminder` | Daily reminder for unsigned docs |
| `document-completed` | All recipients signed, sealed |
| `document-rejected` | Recipient rejected document |
| `welcome` | New user signup |
| `verify-email` | Email verification |
| `password-reset` | Password reset request |
| `team-invitation` | Invited to join team |

All emails are sent via background jobs (never inline in request handling) to avoid blocking responses.

## Cross-References

- Job database model: @SCHEMA.md#backgroundjob
- Provider configuration: @INFRA.md#swappable-providers
- Job error handling: @ERRORS.md#retry-policy
- Job logging: @LOGGING.md#background-job-logging
- Webhook schema: @SCHEMA.md#webhook
- PDF sealing security: @SECURITY.md#pdf-cryptographic-signing
- Email provider options: @INFRA.md#email-provider-interface
