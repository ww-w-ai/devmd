---
devmd: operations
version: "1.0"
project: Twenty CRM

runbook_scope:
  - multi-tenant-database
  - email-calendar-sync
  - worker-queues
  - clickhouse-pipeline
  - backup-recovery
  - workspace-management
  - performance-monitoring

on_call:
  primary: "ops team"
  escalation: "engineering lead"
  sla:
    p1_response: 15min
    p2_response: 1h
    p3_response: 24h
---

# Twenty CRM — Operations

Runbooks and procedures for operating Twenty in production. Covers multi-tenant database management, sync monitoring, queue health, backup/recovery, and performance observability.

## 1. Multi-Tenant Database Operations

Each workspace has a dedicated PostgreSQL schema. See @SCHEMA.md#multi-tenant.

### Schema Management

```sql
-- List all workspace schemas
SELECT schema_name
FROM information_schema.schemata
WHERE schema_name LIKE 'workspace_%'
ORDER BY schema_name;

-- Count tables per workspace schema
SELECT schemaname, count(*) as table_count
FROM pg_tables
WHERE schemaname LIKE 'workspace_%'
GROUP BY schemaname
ORDER BY table_count DESC;

-- Check schema size
SELECT schema_name,
       pg_size_pretty(sum(pg_total_relation_size(schemaname || '.' || tablename))) as total_size
FROM pg_tables
JOIN information_schema.schemata ON schema_name = schemaname
WHERE schemaname LIKE 'workspace_%'
GROUP BY schema_name
ORDER BY sum(pg_total_relation_size(schemaname || '.' || tablename)) DESC;
```

### Connection Pool Monitoring

```sql
-- Active connections per workspace
SELECT usename, state, query, age(clock_timestamp(), query_start) as duration
FROM pg_stat_activity
WHERE datname = 'twenty'
  AND state = 'active'
ORDER BY duration DESC;

-- Connection pool utilization
SELECT count(*) as total,
       count(*) FILTER (WHERE state = 'active') as active,
       count(*) FILTER (WHERE state = 'idle') as idle
FROM pg_stat_activity
WHERE datname = 'twenty';
```

**Alert thresholds:**
- Active connections > 80% of `PG_MAX_CONNECTIONS`: WARN
- Active connections > 95% of `PG_MAX_CONNECTIONS`: CRITICAL
- Idle connections > 50% of total: investigate connection leaks

### Slow Query Detection

```sql
-- Queries running longer than 30 seconds
SELECT pid, usename, query, age(clock_timestamp(), query_start) as duration
FROM pg_stat_activity
WHERE state = 'active'
  AND query_start < now() - interval '30 seconds'
  AND query NOT LIKE '%pg_stat_activity%'
ORDER BY duration DESC;

-- Kill long-running query (with caution)
SELECT pg_terminate_backend(pid);
```

See @LOGGING.md for `db_query_duration_seconds` histogram alerts.

### Lock Monitoring

```sql
-- Detect lock contention
SELECT blocked.pid AS blocked_pid,
       blocked.query AS blocked_query,
       blocking.pid AS blocking_pid,
       blocking.query AS blocking_query
FROM pg_locks AS blocked_locks
JOIN pg_stat_activity AS blocked ON blocked.pid = blocked_locks.pid
JOIN pg_locks AS blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
  AND blocking_locks.database = blocked_locks.database
  AND blocking_locks.relation = blocked_locks.relation
  AND blocking_locks.page = blocked_locks.page
  AND blocking_locks.tuple = blocked_locks.tuple
  AND blocking_locks.pid != blocked_locks.pid
JOIN pg_stat_activity AS blocking ON blocking.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

---

## 2. Email/Calendar Sync Monitoring

See @RUNTIME.md#email-calendar-sync, @FLOWS.md#email-sync-flow.

### Sync Health Dashboard

```sql
-- Connected accounts status overview
SELECT
  provider,
  count(*) as total,
  count(*) FILTER (WHERE "authFailedAt" IS NULL) as healthy,
  count(*) FILTER (WHERE "authFailedAt" IS NOT NULL) as auth_failed,
  min("updatedAt") as oldest_sync,
  max("updatedAt") as latest_sync
FROM workspace_*.connected_account
GROUP BY provider;
```

Note: This query requires cross-schema access. In practice, use the admin API or a monitoring query that iterates workspace schemas.

### Common Sync Failures

| Error | Cause | Resolution |
|---|---|---|
| `IMAP_AUTH_FAILED` | OAuth token expired, refresh failed | User must re-authenticate in Settings > Accounts |
| `IMAP_RATE_LIMITED` | Too many IMAP connections (Gmail: 15 concurrent) | Reduce sync frequency, check for stuck connections |
| `IMAP_NETWORK_ERROR` | DNS failure, timeout | Check server network, retry automatic |
| `IMAP_MAILBOX_NOT_FOUND` | User deleted mailbox or label | Log warning, skip mailbox |
| `PARTICIPANT_MATCH_AMBIGUOUS` | Multiple Person records match same email | Manual merge of duplicate Person records |

### Sync Recovery Procedure

```bash
# 1. Check sync job status in BullMQ
# Use BullMQ dashboard or Redis CLI
redis-cli LLEN "bull:email-sync:waiting"
redis-cli LLEN "bull:email-sync:active"
redis-cli LLEN "bull:email-sync:failed"

# 2. For stuck jobs, drain and re-enqueue
# Via twenty-server CLI:
nx run twenty-server:command -- sync:reset --accountId=<uuid>

# 3. For auth-failed accounts
# No server-side fix — user must re-authenticate via UI
# Admin can view affected accounts:
nx run twenty-server:command -- sync:status --workspace=<id>
```

---

## 3. BullMQ Queue Monitoring

Nine queues power async processing. See @RUNTIME.md#bullmq-worker.

### Queue Inventory

| Queue | Concurrency | Timeout | Retries | Purpose |
|---|---|---|---|---|
| `email-sync` | 5 | 5 min | 3 | IMAP email sync per account |
| `calendar-sync` | 5 | 5 min | 3 | CalDAV calendar sync |
| `webhook-dispatch` | 10 | 30 sec | 3 | Outbound webhook delivery |
| `workflow-execution` | 5 | 2 min | 2 | Workflow action execution |
| `workspace-provisioning` | 2 | 5 min | 1 | New workspace creation |
| `record-indexing` | 10 | 1 min | 2 | Full-text search indexing |
| `analytics-aggregation` | 2 | 10 min | 1 | ClickHouse data pipeline |
| `token-cleanup` | 1 | 2 min | 1 | Expired token purging |
| `trash-cleanup` | 1 | 10 min | 1 | Soft-deleted record purging |

### Monitoring Commands

```bash
# Queue depths (Redis CLI)
for queue in email-sync calendar-sync webhook-dispatch workflow-execution workspace-provisioning record-indexing analytics-aggregation token-cleanup trash-cleanup; do
  echo "$queue: waiting=$(redis-cli LLEN bull:$queue:waiting) active=$(redis-cli LLEN bull:$queue:active) failed=$(redis-cli ZCARD bull:$queue:failed)"
done
```

### Alert Thresholds

| Metric | WARN | CRITICAL | Action |
|---|---|---|---|
| Queue waiting > 100 | Yes | — | Scale workers up |
| Queue waiting > 500 | — | Yes | Investigate backlog cause |
| Failed jobs > 10/hour | Yes | — | Check error logs |
| Failed jobs > 50/hour | — | Yes | Page on-call |
| Job processing time > 2x median | Yes | — | Check database performance |
| Dead letter queue > 0 | Yes | — | Investigate failed jobs |

### Dead Letter Queue Recovery

```bash
# List dead-letter jobs
redis-cli ZRANGEBYSCORE "bull:webhook-dispatch:failed" -inf +inf WITHSCORES

# Retry failed jobs (via twenty-server CLI)
nx run twenty-server:command -- queue:retry --queue=webhook-dispatch --count=10

# Purge old failed jobs
nx run twenty-server:command -- queue:clean --queue=webhook-dispatch --grace=604800
```

---

## 4. ClickHouse Data Pipeline

Optional analytics store. See @SCHEMA.md#clickhouse-analytics.

### Pipeline Health

```sql
-- ClickHouse: check recent data ingestion
SELECT
  toDate(event_time) as day,
  count() as events,
  uniqExact(workspace_id) as workspaces
FROM events
WHERE event_time > now() - INTERVAL 7 DAY
GROUP BY day
ORDER BY day;

-- Check for data lag
SELECT
  max(event_time) as latest_event,
  dateDiff('minute', max(event_time), now()) as lag_minutes
FROM events;
```

**Alert thresholds:**
- Data lag > 10 minutes: WARN
- Data lag > 1 hour: CRITICAL
- No data in 6 hours: CRITICAL (pipeline broken)

### Common Issues

| Issue | Symptoms | Resolution |
|---|---|---|
| Pipeline stalled | No new events in ClickHouse | Check `analytics-aggregation` BullMQ queue |
| Disk full | Insert failures | Expand ClickHouse storage, run TTL cleanup |
| Slow queries | Dashboard timeouts | Check materialized views, optimize queries |

---

## 5. Backup Strategy

See @INFRA.md#backup-strategy.

### PostgreSQL Backup

```bash
# Logical backup (daily, automated via CronJob)
pg_dump -U twenty -Fc twenty > /backups/twenty_$(date +%F_%H%M).dump

# Verify backup integrity
pg_restore --list /backups/twenty_2026-05-13_0300.dump | head -20

# Point-in-time recovery setup (WAL archiving)
# postgresql.conf:
#   archive_mode = on
#   archive_command = 'cp %p /wal_archive/%f'
#   wal_level = replica
```

### Backup Schedule

| Method | Frequency | Retention | Storage | Verification |
|---|---|---|---|---|
| pg_dump (full) | Daily 3 AM UTC | 30 days | S3 / GCS | Monthly restore test |
| WAL archiving | Continuous | 7 days | S3 / GCS | Weekly PITR test |
| Schema export | On metadata change | 90 days | Git / S3 | Automated validation |
| Redis RDB | Hourly | 24 hours | Local + S3 | On-demand |
| ClickHouse | Daily | 30 days | S3 / GCS | Monthly |

### Restore Procedure

```bash
# 1. Stop application
docker compose stop twenty-server twenty-worker

# 2. Restore from logical backup
pg_restore -U twenty -d twenty --clean --if-exists /backups/twenty_2026-05-13_0300.dump

# 3. Or: Point-in-time recovery
# Stop PostgreSQL
# Copy base backup to data directory
# Set recovery_target_time in postgresql.conf
# Start PostgreSQL (replays WAL to target time)

# 4. Restart application
docker compose start twenty-server twenty-worker

# 5. Verify
curl -s http://localhost:3000/healthz
# Check workspace count, recent records
```

---

## 6. Workspace Deletion and Data Cleanup

### Workspace Deletion Procedure

```
Admin requests workspace deletion
  │
  ├── Step 1: Soft-delete workspace record
  │   │  Set workspace.deletedAt = now()
  │   │  Workspace inaccessible to users immediately
  │   └── Grace period: 30 days (can be restored)
  │
  ├── Step 2: After 30-day grace period (cron: trash-cleanup queue)
  │   │  DROP SCHEMA workspace_{id} CASCADE
  │   │  Delete all metadata records for this workspace
  │   │  Delete workspace_member records
  │   │  Delete feature_flag records
  │   │  Delete billing records
  │   └── Physical workspace record deletion
  │
  └── Step 3: Cleanup external resources
      ├── Revoke all OAuth tokens (connected accounts)
      ├── Delete files from storage (S3 or local)
      └── Delete ClickHouse data for this workspace
```

### Data Retention Summary

| Data Type | Retention | Purge Method |
|---|---|---|
| Active records | Indefinite | User-initiated delete |
| Soft-deleted records | 30 days in Trash | `trash-cleanup` cron job |
| Deleted workspaces | 30-day grace period | `trash-cleanup` cron job |
| Expired tokens | Until cleanup job | `token-cleanup` cron (every 6h) |
| Application logs | 30 days | Log rotation / Loki TTL |
| Traces | 7 days | Tempo TTL |
| Metrics | 90 days | Prometheus/Grafana TTL |
| Backups | 30 days (full), 7 days (WAL) | S3 lifecycle rules |

---

## 7. Performance Monitoring

### Sentry Integration

See @ERRORS.md#sentry-integration.

**Backend alerting rules:**
- Error rate > 1% of requests: WARN
- Error rate > 5% of requests: CRITICAL
- P99 latency > 5 seconds: WARN
- P99 latency > 10 seconds: CRITICAL
- Unhandled exception spike (>10 in 5 minutes): CRITICAL

**Frontend alerting rules:**
- LCP > 2.5 seconds: WARN
- LCP > 4 seconds: CRITICAL
- Error rate > 0.5%: WARN
- Session replay triggered (on error): auto-captured

### OpenTelemetry Metrics

See @LOGGING.md#key-metrics for the full metrics list.

**Key operational metrics and their thresholds:**

| Metric | WARN | CRITICAL |
|---|---|---|
| `http_request_duration_seconds` P95 | > 1s | > 5s |
| `graphql_operation_duration_seconds` P95 | > 500ms | > 3s |
| `db_query_duration_seconds` P95 | > 200ms | > 1s |
| `db_connection_pool_size` | > 80% max | > 95% max |
| `cache_hit_ratio` | < 80% | < 50% |
| `worker_job_waiting_seconds` P95 | > 30s | > 5min |
| `email_sync_messages_total` (rate) | drop > 50% | drop > 90% |

### Health Check Endpoints

```bash
# Application health
GET /healthz
# Returns: 200 OK with { status: "healthy", database: "connected", redis: "connected" }
# Returns: 503 if any dependency is unhealthy

# Readiness (for Kubernetes)
GET /readyz
# Returns: 200 when server is ready to accept traffic
# Returns: 503 during startup migrations or shutdown
```

### Incident Response Checklist

```
1. [ ] Acknowledge alert in PagerDuty/Slack
2. [ ] Check health endpoint: curl /healthz
3. [ ] Check Grafana dashboards (API Overview, Database, Worker)
4. [ ] Check Sentry for new errors
5. [ ] Check BullMQ queue depths
6. [ ] Check PostgreSQL connections and locks
7. [ ] Check Redis memory usage
8. [ ] Identify root cause
9. [ ] Apply fix or rollback
10. [ ] Verify recovery via health endpoint + metrics
11. [ ] Write postmortem (if P1/P2)
```

---

## Scheduled Maintenance Windows

| Task | Schedule | Duration | Impact |
|---|---|---|---|
| PostgreSQL VACUUM ANALYZE | Daily 4 AM UTC | ~5 min | None (runs concurrently) |
| Database backup | Daily 3 AM UTC | ~10 min | None |
| Dependency security scan | Weekly Monday 2 AM | ~5 min | None (CI only) |
| Expired token cleanup | Every 6 hours | < 1 min | None |
| Soft-delete purge | Daily 3 AM UTC | < 5 min | None |
| Certificate renewal | Auto (cert-manager) | 0 downtime | None |

## Cross-References

- @SCHEMA.md — Multi-tenant architecture, workspace schema structure
- @RUNTIME.md — BullMQ queues, email/calendar sync, cron jobs
- @INFRA.md — Infrastructure topology, backup configuration
- @LOGGING.md — Metrics, traces, Grafana dashboards
- @ERRORS.md — Error codes, Sentry integration
- @CONFIG.md — Environment variables for monitoring tools
- @DEVOPS.md — CI/CD pipeline, Kubernetes deployment
- @LIFECYCLE.md — State machines for sync, provisioning, records
- @SECURITY.md — Data encryption, access control for operations
