---
devmd: runtime
version: 0.1.0

jobs:
  - name: ""
    type: ""                     # queue | cron | webhook | event
    schedule: ""                 # cron expression if type=cron
    handler: ""                  # function or module path
    concurrency: 1
    timeout: ""
    retry:
      max_attempts: 3
      backoff: ""                # exponential | linear | fixed
      delay_ms: 1000

cron:
  - name: ""
    schedule: ""                 # e.g. "0 */6 * * *"
    handler: ""
    enabled: true

webhooks:
  - name: ""
    path: ""
    method: ""                   # POST | PUT
    auth: ""                     # hmac | bearer | none
    handler: ""
    events: []                   # which events trigger this

email:
  provider: ""                   # sendgrid | ses | resend | ...
  templates:
    - name: ""
      trigger: ""
      subject_template: ""

queues:
  - name: ""
    provider: ""                 # sqs | rabbitmq | redis | cf-queues | ...
    dead_letter: ""
    max_retries: 0
    visibility_timeout: ""
---

# RUNTIME.md

> Job queues, cron schedules, webhook dispatch, email sending, and retry policy.

## Jobs

<!-- Background job definitions. Reference @ARCHITECTURE.md for handler locations. -->

| Job | Type | Schedule | Handler | Timeout |
|-----|------|----------|---------|---------|
|     |      |          |         |         |

## Cron Schedules

<!-- Recurring tasks. Reference @OPERATIONS.md for monitoring. -->

## Webhooks

<!-- Inbound webhook handlers. Reference @API.md for endpoint definitions. -->

## Email

<!-- Transactional email config. Reference @BRAND.md for copy rules. -->

## Queues

<!-- Message queue config. Reference @INFRA.md for queue infrastructure. -->

## Retry Policy

<!-- Default retry behavior. Reference @ERRORS.md for retryable errors. -->

## Cross-References

- Agent execution: @HARNESS.md#lifecycle
- Infrastructure: @INFRA.md
- Error handling: @ERRORS.md
