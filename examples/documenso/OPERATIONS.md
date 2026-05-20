---
devmd: operations
version: "1.0"
project: documenso
updated: 2026-05-13

compliance:
  soc2: true
  gdpr: true

monitoring:
  logs: pino-json-stdout
  analytics: posthog
  apm: datadog
  alerts: [slack, email]

sla:
  uptime: "99.9%"
  signing_latency_p99: "< 5s"
  seal_latency_p99: "< 30s"
---

# OPERATIONS.md — Documenso

## SOC2 Compliance Requirements

Documenso's managed cloud platform is SOC2 Type II certified. Self-hosted instances are responsible for their own compliance.

### Controls Maintained

| Control Area | Implementation |
|---|---|
| Access Control | RBAC (org/team roles). See @SECURITY.md#authorization |
| Authentication | MFA (TOTP + Passkeys), session management. See @SECURITY.md#authentication |
| Audit Logging | 18 event types, append-only, IP + timestamp on every action. See @LOGGING.md#audit-events |
| Data Encryption | TLS in transit, filesystem-level at rest, application-level for secrets. See @SECURITY.md#data-protection |
| Change Management | PR reviews required, CI gates, conventional commits. See @TESTING.md#ci-pipeline |
| Incident Response | Documented procedures (this file) |
| Vulnerability Management | CodeQL weekly scans, Dependabot, 48h response SLA. See @SECURITY.md#vulnerability-reporting |
| Backup & Recovery | Documented procedures (this file) |

### Audit Trail Retention

- Document audit logs: Retained for the lifetime of the document (never purged)
- Application logs: 90 days in log aggregator
- Session records: Expired sessions cleaned daily (see @RUNTIME.md#cron-jobs)

---

## Incident Response

### Severity Levels

| Level | Definition | Response Time | Examples |
|---|---|---|---|
| **P1 — Critical** | Service fully unavailable or data integrity at risk | 15 minutes | Database down, signing service unreachable, data breach |
| **P2 — High** | Core functionality degraded | 1 hour | Seal failures > 5%, email delivery < 90%, auth failures |
| **P3 — Medium** | Non-core functionality impacted | 4 hours | Analytics down, webhook delivery degraded, slow dashboard loads |
| **P4 — Low** | Minor issue, workaround available | Next business day | UI glitch, incorrect translation, non-critical cron job failure |

### Incident Workflow

```
1. Detect (monitoring alert or user report)
2. Triage → assign severity level
3. Acknowledge in Slack #incidents
4. Investigate → identify root cause
5. Mitigate → apply temporary fix or rollback
6. Resolve → deploy permanent fix
7. Post-mortem → document in incident log (P1/P2 only)
```

---

## Signing Failure Runbook

PDF cryptographic sealing is the most critical operation. See @RUNTIME.md#seal-document and @SECURITY.md#sealing-process.

### Symptoms

- `seal-document` jobs accumulating in FAILED status
- Documents stuck in PENDING after all recipients completed
- Error logs: `DOCUMENT_SEAL_FAILED` or `SIGNING_FAILED`

### Diagnosis

```bash
# Check failed seal jobs (last 24h)
SELECT id, payload, "retryCount", "lastRun", "createdAt"
FROM "BackgroundJob"
WHERE name = 'seal-document' AND status = 'FAILED'
AND "createdAt" > NOW() - INTERVAL '24 hours'
ORDER BY "createdAt" DESC;

# Check signing provider health
# Local: verify certificate file exists and is not expired
openssl pkcs12 -in /certs/signing.p12 -nokeys -passin pass:$PASSPHRASE | openssl x509 -noout -enddate

# HSM: verify GCP credentials and key access
gcloud kms keys describe $KEY_NAME --keyring=$KEYRING --location=$LOCATION
```

### Resolution Steps

1. **Certificate expired**: Generate or obtain new certificate. Update `NEXT_PRIVATE_SIGNING_LOCAL_FILE_CONTENTS` or `FILE_PATH`. Restart application.
2. **HSM credential expired**: Rotate GCP service account key. Update `GCLOUD_APPLICATION_CREDENTIALS_CONTENTS`.
3. **PDF corruption**: Check source PDF in storage. Re-upload if corrupted. Manually trigger re-seal.
4. **Memory/timeout**: Seal job timed out on large PDF. Increase `timeout` for `seal-document` job or optimize PDF processing.

### Manual Re-Seal

```bash
# Re-queue failed seal jobs
UPDATE "BackgroundJob"
SET status = 'PENDING', "retryCount" = 0
WHERE name = 'seal-document' AND status = 'FAILED'
AND "createdAt" > NOW() - INTERVAL '24 hours';
```

---

## Email Delivery Monitoring

### Health Indicators

| Metric | Healthy | Warning | Critical |
|---|---|---|---|
| Delivery rate | > 98% | 90-98% | < 90% |
| Bounce rate | < 2% | 2-5% | > 5% |
| `send-signing-email` failure rate | < 1% | 1-5% | > 5% |
| Email send latency (p99) | < 5s | 5-15s | > 15s |

### Diagnosis

```bash
# Check failed email jobs
SELECT name, status, COUNT(*)
FROM "BackgroundJob"
WHERE name LIKE 'send-%email%'
AND "createdAt" > NOW() - INTERVAL '24 hours'
GROUP BY name, status;

# Check recipient send status
SELECT "sendStatus", COUNT(*)
FROM "Recipient"
WHERE "createdAt" > NOW() - INTERVAL '24 hours'
GROUP BY "sendStatus";
```

### Common Issues

| Issue | Cause | Fix |
|---|---|---|
| All emails failing | SMTP credentials expired or provider down | Verify SMTP config. Check provider status page. Switch provider if needed (see @CONFIG.md#provider-selection) |
| High bounce rate | Sending from unverified domain | Configure SPF, DKIM, DMARC for sender domain |
| Emails in spam | Domain reputation issue | Check blacklist status. Warm up new sending domain gradually |
| Inbucket not receiving (dev) | Docker container not running | `docker compose up -d inbucket` |

---

## Database Backup and Recovery

### Managed Cloud (Production)

- Automated daily backups with 30-day retention
- Point-in-time recovery (PITR) within retention window
- Cross-region backup replication

### Self-Hosted

```bash
# Manual backup
pg_dump -h localhost -p 54320 -U documenso -Fc documenso > backup_$(date +%Y%m%d_%H%M%S).dump

# Automated backup (add to crontab)
0 2 * * * pg_dump -h $DB_HOST -U $DB_USER -Fc $DB_NAME > /backups/documenso_$(date +\%Y\%m\%d).dump

# Restore
pg_restore -h localhost -p 54320 -U documenso -d documenso_restore backup_20260513_020000.dump

# Verify restoration
psql -h localhost -p 54320 -U documenso -d documenso_restore -c "SELECT COUNT(*) FROM \"Envelope\";"
```

### Before Migration Safety Protocol

Per global rules (see @CLAUDE.md):

1. **Always backup before any DB operation**
2. **Test migration locally first**: `npx prisma migrate dev` on local DB
3. **Never apply directly to production without staging verification**
4. **Investigate data loss immediately — never shrug off**

---

## Rate Limit Monitoring

See @SECURITY.md#rate-limiting for limit definitions.

### Indicators

| Metric | Healthy | Investigate |
|---|---|---|
| 429 responses/hour | < 10 | > 50 (possible abuse or misconfigured client) |
| Unique IPs hitting limits | < 5 | > 20 (possible DDoS) |
| Authenticated user limit hits | < 5/day | > 20/day (aggressive API usage — contact user) |

### Check Current Rate Limit State

```bash
# Count 429 responses in last hour (from structured logs)
# Filter Pino JSON logs for statusCode=429
cat /var/log/documenso/*.log | jq 'select(.statusCode == 429)' | wc -l

# Or via PostHog (if configured)
# Event: $pageview with property status_code=429
```

---

## Certificate Expiry Management

### Monitoring

| Certificate | Check Command | Renewal Lead Time |
|---|---|---|
| TLS (web) | Managed by hosting provider or cert-manager | Auto-renewed (Let's Encrypt) |
| Signing P12 | `openssl pkcs12 -in cert.p12 -nokeys \| openssl x509 -noout -enddate` | 30 days before expiry |
| HSM Key | `gcloud kms keys describe ...` | Per GCP key rotation policy |

### Renewal Procedure (Local P12)

1. Obtain new certificate from CA
2. Convert to PKCS#12 format: `openssl pkcs12 -export -in cert.pem -inkey key.pem -out new-signing.p12`
3. Base64 encode: `base64 -i new-signing.p12`
4. Update `NEXT_PRIVATE_SIGNING_LOCAL_FILE_CONTENTS` in environment
5. Restart application
6. Verify: send and seal a test document
7. Archive old certificate securely

---

## Cron Job Monitoring

See @RUNTIME.md#cron-jobs for job definitions.

| Job | Expected | Missed Alert After |
|---|---|---|
| `seal-documents` | Every 5 min | 15 min (3 missed runs) |
| `send-reminder-emails` | Daily 9:00 UTC | 10:00 UTC |
| `expire-documents` | Hourly | 2 hours |
| `cleanup-expired-sessions` | Daily 2:00 UTC | 4:00 UTC |
| `process-webhooks` | Every 1 min | 5 min |

### Verification

```bash
# Check last successful run per cron job
SELECT name, MAX("completedAt") as last_success
FROM "BackgroundJob"
WHERE name IN ('seal-documents', 'send-reminder-emails', 'expire-documents',
               'cleanup-expired-sessions', 'process-webhooks')
AND status = 'COMPLETED'
GROUP BY name;
```

---

## Health Check Endpoints

| Endpoint | Purpose | Expected Response |
|---|---|---|
| `GET /health` | Application liveness | `200 OK` |
| `GET /health/ready` | Application readiness (DB connected) | `200 OK` with `{"db": "ok"}` |

Configure load balancer and container orchestrator to use `/health/ready` for routing decisions.

## Cross-References

- Signing security and sealing process: @SECURITY.md#pdf-cryptographic-signing
- Background job system: @RUNTIME.md
- Error codes for operations: @ERRORS.md
- Log format and audit events: @LOGGING.md
- Environment configuration: @CONFIG.md
- CI/CD pipeline: @DEVOPS.md
- Database schema: @SCHEMA.md
