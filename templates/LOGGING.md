---
devmd: logging
version: 0.1.0

log_levels:
  - level: debug
    when: ""
  - level: info
    when: ""
  - level: warn
    when: ""
  - level: error
    when: ""
  - level: fatal
    when: ""

structured_format:
  type: ""                       # json | logfmt | clf
  required_fields:
    - timestamp
    - level
    - message
    - trace_id
    - service

tracing:
  header: ""                     # e.g. x-trace-id
  propagation: ""                # w3c | b3 | custom
  sampling_rate: 1.0

pii_masking:
  fields: []                     # e.g. [email, phone, ip]
  strategy: ""                   # redact | hash | mask

audit_events:
  - event: ""
    trigger: ""
    retention_days: 0
---

# LOGGING.md

> Log levels, structured format, request tracing, audit events, and PII masking.

## Log Levels

<!-- When to use each level. Reference @ERRORS.md#error-codes for error-level mapping. -->

## Structured Format

<!-- Required fields, format type. -->

```json
{"timestamp":"","level":"","message":"","trace_id":"","service":""}
```

## Request Tracing

<!-- Trace ID propagation, sampling. Reference @INFRA.md for observability stack. -->

## Audit Events

<!-- Business events that must be logged. Reference @SECURITY.md#rbac. -->

| Event | Trigger | Retention | PII |
|-------|---------|-----------|-----|
|       |         |           |     |

## PII Masking

<!-- Which fields are masked, strategy. Reference @SECURITY.md for compliance. -->
