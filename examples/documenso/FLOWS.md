---
devmd: flows
version: "1.0"
project: documenso
updated: 2026-05-13

flows:
  - name: document-signing
    type: core
    actors: [sender, recipient, system]
    entry: "Create envelope"
    exit: "Sealed PDF delivered"
    steps: 12

  - name: template-to-document
    type: secondary
    actors: [template-owner, recipient, system]
    entry: "Create template"
    exit: "Document created and signing initiated"

  - name: oauth-login
    type: auth
    actors: [user, browser, oauth-provider, auth-server]
    entry: "Click OAuth provider"
    exit: "Authenticated session"

  - name: webhook-dispatch
    type: integration
    actors: [system, job-worker, external-endpoint]
    entry: "Document event occurs"
    exit: "Webhook delivered or exhausted retries"
---

# FLOWS.md — Documenso

## Document Signing Flow (Core)

The primary end-to-end flow. Every other flow feeds into or extends this one.

### UX + Data Side-by-Side

| Step | UX (What User Sees) | Data (What System Does) |
|---|---|---|
| 1. Create Envelope | Sender clicks "New Document" on dashboard | `INSERT Envelope(status=DRAFT, envelopeType=DOCUMENT)` + `INSERT DocumentData` |
| 2. Upload PDF | Drag-and-drop or file picker. PDF preview renders in center panel | Store PDF via storage provider (DB or S3). Create `EnvelopeItem` linking to `DocumentData` |
| 3. Add Recipients | Sidebar form: name, email, role (SIGNER/APPROVER/VIEWER/CC/ASSISTANT) | `INSERT Recipient` per entry. Assign signing order if sequential. See @GLOSSARY.md#role-permissions |
| 4. Place Fields | Drag fields from palette onto PDF pages. Assign each field to a recipient (color-coded) | `INSERT Field` with page number, x/y coordinates, width/height, type, recipientId. See @GLOSSARY.md#field-type-reference |
| 5. Configure Options | Set document title, message, redirect URL, expiry, auth requirements | `UPDATE Envelope` metadata. `UPDATE DocumentMeta` for signing options |
| 6. Send | Click "Send" button. Confirmation dialog shows recipient summary | Validate: >= 1 recipient, >= 1 field per signer. `UPDATE Envelope(status=PENDING)`. Write `DOCUMENT_SENT` audit log. Enqueue `send-signing-email` job per recipient. See @RUNTIME.md#send-signing-email |
| 7. Email Received | Recipient receives email with "Review Document" CTA button | Email template rendered via `@documenso/email`. Contains unique signing token URL (`/sign/{token}`) |
| 8. Open Signing View | Recipient clicks link. Sees document title, sender info, field count | Load envelope + fields + recipient by token. Write `DOCUMENT_OPENED` audit log. Validate recipient auth if configured |
| 9. Fill Fields | Sequential field navigation. Each field type renders its input (text box, date picker, dropdown, etc.) | `UPDATE Field` with inserted value. Write `DOCUMENT_FIELD_INSERTED` audit log per field |
| 10. Sign | Signature pad opens (type/draw/upload tabs). Recipient draws or types signature | `INSERT Signature` with base64 image. Associate with signature field. See @UI.md#signature-pad-component |
| 11. Confirm | Summary of all completed fields. Click "Complete Signing". Legal disclosure shown | `UPDATE Recipient(signingStatus=SIGNED)`. Write `DOCUMENT_RECIPIENT_COMPLETED` audit log. Check if all required recipients completed |
| 12. Seal & Complete | Recipient sees "Document Complete" confirmation. Download link available (if all done) | If all recipients done: enqueue `seal-document` job (critical priority). PDF gets cryptographic signature embedded. `UPDATE Envelope(status=COMPLETED)`. Enqueue `send-completion-email` to all parties. See @RUNTIME.md#seal-document, @SECURITY.md#sealing-process |

### Flow Diagram

```
Sender                          System                          Recipient
  │                                │                                │
  ├── Create Envelope ────────────►│                                │
  ├── Upload PDF ─────────────────►│                                │
  ├── Add Recipients ─────────────►│                                │
  ├── Place Fields ───────────────►│                                │
  ├── Send ───────────────────────►│                                │
  │                                ├── Validate ─────────┐          │
  │                                ├── Status → PENDING   │          │
  │                                ├── Enqueue emails ───►│          │
  │                                │                      ▼          │
  │                                │              ┌── Email ────────►│
  │                                │              │                  ├── Open link
  │                                │◄─────────────┤                  ├── Review PDF
  │                                │              │                  ├── Fill fields
  │                                │              │                  ├── Draw signature
  │                                │◄── Complete ─┤                  ├── Confirm
  │                                │              │                  │
  │                                ├── All done? ─┤                  │
  │                                │   YES ───────┤                  │
  │                                ├── Seal PDF   │                  │
  │                                ├── Embed sig  │                  │
  │                                ├── COMPLETED  │                  │
  │                                ├── Email all ─┴─────────────────►│
  │◄── Completion email ──────────┤                                  │
```

### Error Paths

| At Step | Error | Recovery |
|---|---|---|
| 6. Send | Missing recipients or fields | Validation error shown. Sender corrects and retries. See @ERRORS.md#document-lifecycle-errors |
| 7. Email | Email delivery fails | Job retried 3x with backoff. See @RUNTIME.md#retry-policy |
| 8. Open | Token expired or invalid | Recipient sees error page. Sender can resend via `document.resendDocument` |
| 11. Confirm | Recipient rejects | `UPDATE Recipient(signingStatus=REJECTED)`. `UPDATE Envelope(status=REJECTED)`. Notify sender. See @LIFECYCLE.md#recipient-signing |
| 12. Seal | Cryptographic seal fails | `seal-document` retried 5x (critical priority). Alert on final failure. See @RUNTIME.md#seal-document |

---

## Template to Document Flow

Templates allow pre-configured envelopes that can be reused or shared via direct links.

### Standard Template Usage

| Step | UX | Data |
|---|---|---|
| 1. Create Template | Navigate to /templates → "New Template". Upload PDF, add placeholder recipients, place fields | `INSERT Envelope(envelopeType=TEMPLATE)`. Same field/recipient structure as documents |
| 2. Configure | Set template title, default message, signing options | `UPDATE Envelope` + `UPDATE DocumentMeta`. Template-specific: external ID, direct link toggle |
| 3. Use Template | Click "Use Template" from template list. Fill in actual recipient emails | `template.createDocumentFromTemplate`: clone Envelope with `envelopeType=DOCUMENT`, copy all items/fields/recipients. New envelope gets `status=DRAFT` |
| 4. Send | Same as document signing flow step 6+ | Continues into @FLOWS.md#document-signing-flow |

### Direct Link Flow

Direct links let anyone initiate signing without sender involvement.

| Step | UX | Data |
|---|---|---|
| 1. Enable Direct Link | Template owner toggles "Direct Link" in template settings | Generate unique direct link token. Store in `TemplateDirectLink` model |
| 2. Share Link | Copy `/d/{token}` URL. Share via email, website, or embed | No system action — link is static |
| 3. Recipient Opens | Recipient navigates to direct link URL | Load template by token. Create new document from template. Auto-assign recipient to signer role |
| 4. Fill & Sign | Recipient fills fields and signs (same as signing flow steps 9-11) | Same field insertion + signature flow. Auto-completes since single signer |
| 5. Seal | System seals document after recipient confirms | Same seal flow. Document owner sees completed document in dashboard |

---

## OAuth Login Flow

See @SECURITY.md#oauth for provider configuration.

| Step | UX | Data |
|---|---|---|
| 1. Click Provider | User clicks "Continue with Google" on /signin | Generate OAuth state + PKCE code verifier. Store in short-lived cookie. Redirect to provider authorization URL |
| 2. Provider Auth | User authenticates with Google (or OIDC provider). Grants permissions | Provider handles authentication. Redirects back with authorization code |
| 3. Callback | Browser redirects to `/auth/oauth/callback?code=...&state=...` | Validate state matches cookie. Exchange code for access token + ID token. Extract email, name, avatar from ID token claims |
| 4. Account Resolution | (Transparent to user) | Find existing user by email → link OAuth identity. Or create new User + OAuthAccount + default Organisation/Team. See @SCHEMA.md#user |
| 5. Session Created | User lands on dashboard, fully authenticated | `INSERT Session(token, userId, expiresAt=+30d)`. Set encrypted `__session` cookie. See @SECURITY.md#session-management |

### Edge Cases

- **Email conflict**: OAuth email matches existing password account → link accounts (user must verify ownership)
- **Disabled provider**: OAuth button hidden. Existing OAuth users can still log in until admin disables
- **OIDC custom provider**: Configured via `NEXT_PRIVATE_OIDC_WELL_KNOWN` env var. Discovery endpoint auto-configures

---

## Webhook Dispatch Flow

See @RUNTIME.md#dispatch-webhook for job details and @API.md#webhook for registration.

| Step | UX | Data |
|---|---|---|
| 1. Event Occurs | (No direct UX — triggered by document lifecycle) | Document event fires (SENT, COMPLETED, REJECTED, etc.). See @RUNTIME.md#webhook-events |
| 2. Find Webhooks | (No UX) | Query `Webhook` table for team's registered webhooks matching event type + enabled status |
| 3. Enqueue Jobs | (No UX) | `enqueueJob('dispatch-webhook', { webhookId, eventType, data })` per matching webhook. Normal priority |
| 4. Deliver | (No UX) | Worker builds JSON payload. Signs with HMAC-SHA256 if secret configured. `POST` to webhook URL with `X-Webhook-Signature` + `X-Webhook-Timestamp` headers |
| 5a. Success | Webhook config shows green status in /settings/webhooks | HTTP 2xx received. Mark delivery as successful |
| 5b. Failure + Retry | Webhook config shows yellow/red status with retry count | Non-2xx or timeout. Retry with exponential backoff: 1min → 5min → 30min. Max 3 retries |
| 5c. Exhausted | Webhook shows failed status. No more retries | All retries failed. Log warning. Webhook remains enabled for future events |

### Payload Example

```json
{
  "event": "DOCUMENT_COMPLETED",
  "timestamp": "2026-05-13T12:00:00Z",
  "data": {
    "envelopeId": 123,
    "documentTitle": "Service Agreement",
    "status": "COMPLETED",
    "recipients": [
      {
        "email": "signer@example.com",
        "role": "SIGNER",
        "status": "SIGNED",
        "completedAt": "2026-05-13T11:58:00Z"
      }
    ]
  }
}
```

## Cross-References

- Envelope state machine: @LIFECYCLE.md#envelope-status
- Recipient states: @LIFECYCLE.md#recipient-signing
- Background job execution: @RUNTIME.md
- Field types and validation: @GLOSSARY.md#field-type-reference
- Signing UI details: @UI.md#signing-flow
- Error handling per flow: @ERRORS.md#document-lifecycle-errors
- PDF sealing security: @SECURITY.md#pdf-cryptographic-signing
