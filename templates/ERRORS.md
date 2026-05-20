---
devmd: errors
version: 0.1.0

error_prefix: ""                 # e.g. "APP" → APP_001

error_codes:
  - code: ""
    http_status: 0
    message: ""
    recovery: ""                 # user-facing recovery hint
    internal_note: ""
    category: ""                 # auth | validation | resource | system | external

exception_hierarchy:
  base: ""                       # e.g. AppError
  children:
    - name: ""
      parent: ""
      default_http: 0

retry_policy:
  retryable_categories: []       # which categories can be retried
  max_retries: 3
  backoff: ""                    # exponential | linear | fixed
  initial_delay_ms: 1000
---

# ERRORS.md

> Error codes, exception hierarchy, and retry policy.

## Error Codes

<!-- One row per error. Reference @API.md#error-format for response shape. -->

| Code | HTTP | Message | Recovery | Category |
|------|------|---------|----------|----------|
|      |      |         |          |          |

## Exception Hierarchy

<!-- Class tree. Reference @ARCHITECTURE.md#layers for where errors are caught. -->

```
AppError
├── AuthError (401/403)
├── ValidationError (400)
├── NotFoundError (404)
├── ConflictError (409)
└── SystemError (500)
```

## Retry Policy

<!-- Which errors are retryable, backoff strategy. Reference @LOGGING.md#audit-events. -->

## User-Facing Messages

<!-- Rules for error copy. Reference @BRAND.md#voice for tone. -->
