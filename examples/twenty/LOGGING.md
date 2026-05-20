---
devmd: logging
version: "1.0"
project: Twenty CRM

log_format: structured_json
log_levels:
  - level: ERROR
    usage: "Unhandled exceptions, data corruption, security violations"
    alert: true
    sentry: true
  - level: WARN
    usage: "Degraded performance, deprecated usage, retry attempts"
    alert: false
    sentry: false
  - level: INFO
    usage: "Request lifecycle, business events, state transitions"
    alert: false
    sentry: false
  - level: DEBUG
    usage: "Detailed execution flow, query plans, cache hits/misses"
    alert: false
    sentry: false
    production: false

structured_fields:
  required:
    - field: timestamp
      format: ISO 8601
    - field: level
      format: "ERROR | WARN | INFO | DEBUG"
    - field: message
      format: "Human-readable description"
    - field: service
      format: "twenty-server | twenty-worker | twenty-front"
  context:
    - field: traceId
      format: "OpenTelemetry trace ID (W3C Trace Context)"
    - field: spanId
      format: "OpenTelemetry span ID"
    - field: workspaceId
      format: UUID
    - field: userId
      format: UUID
    - field: requestId
      format: UUID
  optional:
    - field: operation
      format: "GraphQL operation name"
    - field: duration
      format: "milliseconds"
    - field: errorCode
      format: "See @ERRORS.md"
    - field: metadata
      format: "Arbitrary key-value pairs"

tracing:
  protocol: OpenTelemetry
  propagation: W3C Trace Context
  exporter: OTLP (gRPC to OpenTelemetry Collector)
  sampling:
    production: "10% of requests, 100% of errors"
    staging: "100%"
    development: "100%"
  spans:
    - name: "graphql.operation"
      attributes: [operationName, operationType, workspaceId]
    - name: "graphql.resolve"
      attributes: [fieldName, parentType]
    - name: "db.query"
      attributes: [statement, schema, duration]
    - name: "http.request"
      attributes: [method, url, statusCode, duration]
    - name: "worker.job"
      attributes: [queueName, jobName, attempt, duration]
    - name: "email.sync"
      attributes: [provider, messagesProcessed, duration]

observability_stack:
  collector: OpenTelemetry Collector
  metrics: Grafana (dashboards)
  traces: "Grafana Tempo (or Jaeger)"
  logs: "Grafana Loki (or stdout → log aggregator)"
  errors: Sentry 10
---

# Twenty CRM Logging & Observability

## Application Logs Module

The `application-logs` module in twenty-server provides structured logging across all services.

### Log Format

```json
{
  "timestamp": "2026-05-13T10:30:00.123Z",
  "level": "INFO",
  "message": "GraphQL operation completed",
  "service": "twenty-server",
  "traceId": "abc123def456",
  "spanId": "789ghi",
  "workspaceId": "ws-uuid-123",
  "userId": "user-uuid-456",
  "requestId": "req-uuid-789",
  "operation": "findPeople",
  "duration": 45,
  "metadata": {
    "resultCount": 20,
    "hasNextPage": true,
    "cacheHit": false
  }
}
```

### Log Categories

| Category | Level | Example |
|---|---|---|
| Request lifecycle | INFO | `GraphQL operation started/completed` |
| Authentication | INFO/WARN | `User authenticated`, `Token refresh failed` |
| Database query | DEBUG | `SELECT * FROM workspace_abc.person WHERE ...` |
| Cache | DEBUG | `Cache hit: object_metadata:ws-123`, `Cache miss: field_metadata:ws-123` |
| Worker job | INFO | `Job started: email-sync`, `Job completed: email-sync (45ms)` |
| Webhook dispatch | INFO/WARN | `Webhook sent to https://...`, `Webhook failed: 500 (retry 2/3)` |
| Schema change | INFO | `Object created: project (workspace ws-123)` |
| Error | ERROR | `Unhandled exception in findPeople resolver` |
| Security | WARN/ERROR | `Permission denied: user-456 → READ_PERSON`, `SQL injection attempt blocked` |

## OpenTelemetry Integration

### Trace Propagation

```
Browser (twenty-front)
  │  traceparent: 00-{traceId}-{spanId}-01
  ▼
API Server (twenty-server)
  │  Extracts trace context from headers
  │  Creates child spans for:
  │    - GraphQL operation resolution
  │    - Database queries via twenty-orm
  │    - External HTTP calls
  │    - Cache operations
  ▼
Worker (BullMQ)
  │  Trace context propagated via job metadata
  │  Creates child spans for:
  │    - Job execution
  │    - Email sync operations
  │    - Webhook dispatch
  ▼
OpenTelemetry Collector
  │  Receives traces, metrics, logs via OTLP
  │  Exports to configured backends
  ▼
Grafana (visualization)
```

### Key Metrics

| Metric | Type | Description |
|---|---|---|
| `http_request_duration_seconds` | Histogram | API request latency |
| `graphql_operation_duration_seconds` | Histogram | GraphQL operation latency by name |
| `db_query_duration_seconds` | Histogram | Database query latency |
| `db_connection_pool_size` | Gauge | Active DB connections |
| `cache_hit_ratio` | Gauge | Redis cache hit rate |
| `worker_job_duration_seconds` | Histogram | BullMQ job execution time |
| `worker_job_waiting_seconds` | Histogram | Time in queue before processing |
| `worker_active_jobs` | Gauge | Currently processing jobs |
| `email_sync_messages_total` | Counter | Total messages synced |
| `webhook_delivery_duration_seconds` | Histogram | Webhook HTTP call latency |
| `workspace_count` | Gauge | Total active workspaces |

### Grafana Dashboards

Pre-built dashboards in the infrastructure. See @INFRA.md#observability.

1. **API Overview** — Request rate, latency P50/P95/P99, error rate, top operations
2. **Database** — Query latency, connection pool, slow queries, lock waits
3. **Worker** — Job throughput, queue depth, failure rate, processing time
4. **Email Sync** — Messages synced, sync errors, latency per provider
5. **Business Metrics** — Active users, records created, API calls per workspace

## Frontend Observability

### Sentry Browser SDK

```typescript
Sentry.init({
  dsn: process.env.REACT_APP_SENTRY_DSN,
  release: process.env.REACT_APP_VERSION,
  environment: process.env.NODE_ENV,
  integrations: [
    Sentry.browserTracingIntegration(),
    Sentry.replayIntegration(),
  ],
  tracesSampleRate: 0.1,     // 10% of transactions
  replaysSessionSampleRate: 0, // Only capture replays on error
  replaysOnErrorSampleRate: 1.0,
});
```

### Frontend Performance Tracking

- **Page load** — Core Web Vitals (LCP, FID, CLS) via Sentry
- **GraphQL operations** — Duration tracked per operation name
- **Route transitions** — Navigation timing between routes
- **Component renders** — React Profiler for expensive components (dev only)

## Request Tracing Flow

A complete trace through the system:

```
1. User clicks "Create Person" in UI
2. twenty-front creates span: ui.action (button click)
3. Apollo Client sends mutation with traceparent header
4. twenty-server creates span: http.request (POST /api)
5.   → span: graphql.operation (createOnePerson)
6.   → span: auth.validate (JWT verification)
7.   → span: permission.check (WRITE_PERSON)
8.   → span: db.query (INSERT INTO workspace_abc.person)
9.   → span: event.emit (person.created)
10.  → span: webhook.dispatch (POST https://external.com/hook)
11.  → span: cache.invalidate (person list cache)
12. Response returned with trace ID in headers
13. Worker picks up async jobs (trace propagated via job metadata)
14.  → span: worker.job (index-person-for-search)
```

## Log Retention

| Environment | Retention | Storage |
|---|---|---|
| Production | 30 days (logs), 7 days (traces), 90 days (metrics) | Grafana Cloud or self-hosted Loki/Tempo |
| Staging | 7 days | Same stack, lower tier |
| Development | Session only | stdout |

## Cross-References

- @ERRORS.md — Error codes and Sentry integration
- @INFRA.md — OpenTelemetry Collector and Grafana deployment
- @SECURITY.md — Security event logging
- @RUNTIME.md — Worker job logging
