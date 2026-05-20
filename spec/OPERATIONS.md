# OPERATIONS.md Specification

> Version: 0.1.0-draft | Status: Draft

## Purpose

OPERATIONS.md defines incident response, service level objectives, health checks, runbooks, and escalation procedures. It is the operational counterpart to `@DEVOPS.md` (build/deploy) and `@INFRA.md` (infrastructure intent).

## Frontmatter Schema

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Operations configuration name |
| `severity_levels` | `map<string, SeverityLevel>` | REQUIRED | Incident severity definitions. Min 1. |
| `slos` | `SLO[]` | OPTIONAL | Service level objectives |
| `health_checks` | `HealthCheck[]` | OPTIONAL | Automated health probes |
| `runbooks` | `map<string, Runbook>` | OPTIONAL | Named operational runbooks |

### SeverityLevel

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `response_time` | `string` | REQUIRED | Maximum time to acknowledge (e.g., `15m`, `1h`, `4h`) |
| `description` | `string` | REQUIRED | What constitutes this severity |

### SLO

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `metric` | `string` | REQUIRED | What is measured (e.g., `availability`, `p99_latency`) |
| `target` | `string` | REQUIRED | Target value (e.g., `99.9%`, `<200ms`) |
| `window` | `string` | REQUIRED | Measurement window (e.g., `30d`, `7d`) |

### HealthCheck

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `endpoint` | `string` | REQUIRED | URL or path to probe |
| `expected` | `string` | REQUIRED | Expected response (e.g., `200 OK`, `{"status":"ok"}`) |
| `interval` | `string` | REQUIRED | Check frequency (e.g., `30s`, `5m`) |

### Runbook

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `trigger` | `string` | REQUIRED | Condition or alert that activates this runbook |
| `steps` | `string[]` | REQUIRED | Ordered resolution steps. Min 1. |
| `escalation` | `string` | OPTIONAL | Who to escalate to if steps do not resolve |

## Body Sections

Sections MUST appear in this order when present.

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## Incident Severity` | REQUIRED | Severity level definitions and examples. |
| `## SLOs` | OPTIONAL | SLO rationale, error budget policy. |
| `## Health Checks` | OPTIONAL | Probe details, failure thresholds. |
| `## Runbooks` | REQUIRED | Detailed runbook procedures. |
| `## Escalation` | OPTIONAL | Escalation chain, contact methods, on-call rotation. |
| `## Post-Mortem Template` | OPTIONAL | Template for incident post-mortems. |

## Cross-References

- MUST reference `@LOGGING.md` for alert source definitions and log queries.
- SHOULD reference `@INFRA.md` for infrastructure context in runbooks.
- SHOULD reference `@DEVOPS.md` for rollback procedures.

## Validation Rules

1. `severity_levels` MUST contain at least 1 entry.
2. Every `Runbook.steps` MUST contain at least 1 step.
3. `SLO.window` MUST be a recognized duration format.
4. `HealthCheck.interval` MUST be a recognized duration format.

## Conflict Detection

- Health check endpoints SHOULD correspond to routes defined in `@API.md` or infrastructure in `@INFRA.md`.
- Runbook escalation targets SHOULD be consistent across all runbooks.
- SLO metrics SHOULD align with metrics exposed in `@LOGGING.md#metrics`.
