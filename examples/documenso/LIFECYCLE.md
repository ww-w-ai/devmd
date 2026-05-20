---
devmd: lifecycle
version: "1.0"
project: documenso
updated: 2026-05-13

state_machines:
  - name: envelope-status
    model: Envelope
    field: status
    states: [DRAFT, PENDING, COMPLETED, REJECTED]
    initial: DRAFT
    terminal: [COMPLETED, REJECTED]

  - name: recipient-signing
    model: Recipient
    field: signingStatus
    states: [NOT_SIGNED, SIGNED, REJECTED]
    initial: NOT_SIGNED
    terminal: [SIGNED, REJECTED]

  - name: background-job
    model: BackgroundJob
    field: status
    states: [PENDING, PROCESSING, COMPLETED, FAILED]
    initial: PENDING
    terminal: [COMPLETED, FAILED]

  - name: recipient-send-status
    model: Recipient
    field: sendStatus
    states: [NOT_SENT, SENT, FAILED]
    initial: NOT_SENT
---

# LIFECYCLE.md — Documenso

## Envelope Status

The central state machine governing document and template lifecycle.

### State Diagram

```
                  send()
    ┌───────┐  ────────────►  ┌─────────┐
    │ DRAFT │                 │ PENDING  │
    └───────┘  ◄────────────  └─────────┘
                 (no path       │       │
                  back — by     │       │
                  design)       │       │
                                │       │
                    all signed  │       │  any rejected
                                ▼       ▼
                          ┌──────────┐  ┌──────────┐
                          │COMPLETED │  │ REJECTED │
                          └──────────┘  └──────────┘
                           (terminal)    (terminal)
```

### Transition Rules

| From | To | Trigger | Preconditions | Side Effects |
|---|---|---|---|---|
| `DRAFT` | `PENDING` | `document.sendDocument` | >= 1 recipient with email. >= 1 field per SIGNER/APPROVER. >= 1 EnvelopeItem (uploaded PDF). All field assignments valid | Enqueue `send-signing-email` per recipient. Write `DOCUMENT_SENT` audit log. Fire `DOCUMENT_SENT` webhook. See @RUNTIME.md#send-signing-email |
| `PENDING` | `COMPLETED` | System (auto) | All required recipients have `signingStatus = SIGNED`. Seal job succeeds | Enqueue `seal-document` job. After seal: enqueue `send-completion-email`. Write `DOCUMENT_COMPLETED` audit log. Fire `DOCUMENT_COMPLETED` webhook. See @RUNTIME.md#seal-document |
| `PENDING` | `REJECTED` | `recipient.rejectDocument` | Any single recipient rejects | `UPDATE Recipient(signingStatus=REJECTED)`. Write `DOCUMENT_RECIPIENT_REJECTED` audit log. Fire `DOCUMENT_REJECTED` webhook. Notify sender via email |

### Constraints

- **No PENDING → DRAFT**: Once sent, a document cannot return to draft. The sender must duplicate and re-send.
- **No COMPLETED → any**: Completed documents are immutable. The sealed PDF is the final record.
- **No REJECTED → any**: Rejection is terminal. Sender must create a new document.
- **DRAFT deletion**: Only DRAFT envelopes can be deleted (soft-delete via `deletedAt`).
- **Templates**: Envelopes with `envelopeType = TEMPLATE` stay in `DRAFT` permanently. They are never sent directly — only cloned into DOCUMENT envelopes via `template.createDocumentFromTemplate`.

---

## Recipient Signing

Per-recipient signing status tracks individual progress within a PENDING envelope.

### State Diagram

```
                    completeRecipient()
    ┌────────────┐  ──────────────────►  ┌────────┐
    │ NOT_SIGNED │                       │ SIGNED │
    └────────────┘                       └────────┘
          │                               (terminal)
          │  rejectDocument()
          │
          ▼
    ┌──────────┐
    │ REJECTED │
    └──────────┘
     (terminal)
```

### Transition Rules

| From | To | Trigger | Preconditions | Side Effects |
|---|---|---|---|---|
| `NOT_SIGNED` | `SIGNED` | `recipient.completeRecipient` | All required fields for this recipient have values. Signature field(s) filled if recipient role is SIGNER | Write `DOCUMENT_RECIPIENT_COMPLETED` audit log. Fire `RECIPIENT_COMPLETED` webhook. Check if all recipients complete → trigger envelope COMPLETED transition |
| `NOT_SIGNED` | `REJECTED` | `recipient.rejectDocument` | Recipient has rejection permission (SIGNER, APPROVER roles). Document is PENDING | Write `DOCUMENT_RECIPIENT_REJECTED` audit log. Trigger envelope REJECTED transition. Fire `RECIPIENT_REJECTED` webhook |

### Sequential Signing

When signing order is SEQUENTIAL (see @GLOSSARY.md#signing-order):

- Recipients are ordered by `signingOrder` field
- Recipient N can only access signing view after Recipient N-1 reaches SIGNED status
- Signing emails are sent progressively: next recipient notified only after previous completes
- If any recipient in sequence REJECTS, the entire document is REJECTED

### Role-Based Completion

| Role | What "Complete" Means |
|---|---|
| SIGNER | All assigned fields filled + signature provided |
| APPROVER | Explicit approval action (no signature required) |
| VIEWER | Views document (auto-completes on open, no action needed) |
| CC | Receives copy only (auto-completes, not counted for PENDING → COMPLETED) |
| ASSISTANT | Fills fields on behalf of another recipient |

---

## Recipient Send Status

Tracks email delivery state per recipient.

```
    ┌──────────┐    send-signing-email job succeeds    ┌────────┐
    │ NOT_SENT │  ───────────────────────────────────►  │  SENT  │
    └──────────┘                                        └────────┘
          │
          │  send-signing-email job fails (max retries)
          ▼
    ┌──────────┐
    │  FAILED  │
    └──────────┘
```

- FAILED send status does not block the signing flow — recipient can still access via direct URL
- Sender can manually re-send via `document.resendDocument`

---

## Background Job Status

See @RUNTIME.md for full job execution details.

### State Diagram

```
                   worker claims
    ┌─────────┐  ───────────────►  ┌────────────┐
    │ PENDING │                    │ PROCESSING │
    └─────────┘                    └────────────┘
         ▲                           │         │
         │                           │         │
         │  retry (count < max)      │         │  success
         │                           │         │
         └───────────────────────────┘         ▼
                                          ┌───────────┐
              failure (count >= max)       │ COMPLETED │
                   │                      └───────────┘
                   ▼
              ┌────────┐
              │ FAILED │
              └────────┘
```

### Transition Rules

| From | To | Trigger | Details |
|---|---|---|---|
| `PENDING` | `PROCESSING` | Worker claims job | Worker updates status + sets `lastRun`. Only one worker can claim (DB lock or Redis lock depending on provider) |
| `PROCESSING` | `COMPLETED` | Handler succeeds | Set `completedAt`. Job stays in DB for audit |
| `PROCESSING` | `PENDING` | Handler fails + retries remain | Increment `retryCount`. Schedule retry after backoff delay. See @RUNTIME.md#retry-policy |
| `PROCESSING` | `FAILED` | Handler fails + no retries remain | Set `completedAt`. Log error. Alert if critical priority |

### Priority Retry Limits

| Priority | Max Retries | Backoff Pattern |
|---|---|---|
| Critical (`seal-document`) | 5 | 30s → 1m → 5m → 15m → 30m |
| High (`send-signing-email`) | 3 | 10s → 1m → 5m |
| Normal (`dispatch-webhook`) | 3 | 1m → 5m → 30m |
| Low (`generate-audit-pdf`) | 2 | 5m → 30m |

---

## Development Lifecycle — DevMD File Activity

Which DevMD files are most relevant at each development phase:

### Phase 1: Planning

| File | Activity |
|---|---|
| @PRODUCT.md | Define feature scope, success criteria, target users |
| @GLOSSARY.md | Establish domain terms for new concepts |
| @FLOWS.md | Map end-to-end user journeys |

### Phase 2: Design

| File | Activity |
|---|---|
| @ARCHITECTURE.md | Decide where new code lives (which package, which layer) |
| @SCHEMA.md | Design database models and migrations |
| @API.md | Define tRPC procedures and OpenAPI endpoints |
| @LIFECYCLE.md | Define state machines for new entities |
| @DESIGN.md | Component tokens and variants |
| @SCREENS.md | Visual mockups and screen states |

### Phase 3: Implementation

| File | Activity |
|---|---|
| @CLAUDE.md | Coding conventions, commands, dependency direction |
| @ERRORS.md | Define error codes for new domain |
| @LOGGING.md | Add audit events and structured log points |
| @RUNTIME.md | Define background jobs if needed |
| @CONFIG.md | Add environment variables for new features |

### Phase 4: Testing & QA

| File | Activity |
|---|---|
| @TESTING.md | Write E2E tests following conventions |
| @SECURITY.md | Security review (auth, RBAC, input validation) |
| @DEVOPS.md | CI workflow updates if needed |

### Phase 5: Release

| File | Activity |
|---|---|
| @CHANGELOG.md | Record architectural decision and migration guide |
| @DEVOPS.md | Deployment configuration |
| @OPERATIONS.md | Monitoring and runbook updates |
| @BRAND.md | User-facing copy and messaging |
| @SEO.md | Search visibility for new pages |

## Cross-References

- Envelope and Recipient models: @SCHEMA.md#envelope, @SCHEMA.md#recipient
- Document signing flow: @FLOWS.md#document-signing-flow
- Background job execution: @RUNTIME.md
- Error codes per transition: @ERRORS.md#document-lifecycle-errors
- Webhook events per transition: @RUNTIME.md#webhook-events
- Audit log events: @LOGGING.md#audit-event-types
