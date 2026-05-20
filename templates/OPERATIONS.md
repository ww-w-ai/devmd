---
devmd: operations
version: 0.1.0

severity_levels:
  - level: ""                    # P0 | P1 | P2 | P3
    description: ""
    response_time: ""
    resolution_time: ""
    escalation: ""

slos:
  - metric: ""                   # uptime | latency | error_rate
    target: ""                   # e.g. "99.9%"
    window: ""                   # e.g. "30d"

health_checks:
  - endpoint: ""
    interval: ""
    timeout: ""
    dependencies: []             # what it checks

backup:
  strategy: ""                   # daily | continuous | snapshot
  retention: ""
  restore_tested: ""             # date of last restore test
  location: ""

runbooks:
  - title: ""
    trigger: ""
    steps: []
---

# OPERATIONS.md

> Incident severity, runbooks, backup/restore, SLOs, and health checks.

## Severity Levels

<!-- Incident classification. Reference @ERRORS.md for error-to-severity mapping. -->

| Level | Description | Response | Resolution | Escalation |
|-------|-------------|----------|------------|------------|
| P0    |             |          |            |            |
| P1    |             |          |            |            |
| P2    |             |          |            |            |

## SLOs

<!-- Service level objectives. Reference @INFRA.md#monitoring for tracking. -->

| Metric | Target | Window |
|--------|--------|--------|
|        |        |        |

## Health Checks

<!-- Endpoints and dependencies. Reference @API.md for health endpoint. -->

## Backup & Restore

<!-- Strategy, retention, last tested. Reference @SCHEMA.md for DB backup. -->

## Runbooks

<!-- Step-by-step incident response. Reference @LOGGING.md for investigation. -->

### [Runbook Title]

- **Trigger:**
- **Steps:**
  1.
  2.
