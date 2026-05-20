---
devmd: glossary
version: "1.0"
project: documenso
updated: 2026-05-13

domain: document-signing
methodology: DDD
language: en

terms:
  # Core domain objects
  - term: Envelope
    definition: "Polymorphic container that wraps either a Document or a Template. The top-level entity in the signing workflow."
    type: aggregate-root
    db_model: Envelope
    enum: "EnvelopeType: DOCUMENT | TEMPLATE"
    see_also: [Document, Template]

  - term: Document
    definition: "A specific instance of a PDF uploaded for signing. Always wrapped in an Envelope of type DOCUMENT."
    type: entity
    lifecycle: "DRAFT → PENDING → COMPLETED | REJECTED"
    see_also: [Envelope, DocumentStatus]

  - term: Template
    definition: "A reusable signing blueprint. Wrapped in an Envelope of type TEMPLATE. Used to create Documents with pre-configured recipients and fields."
    type: entity
    see_also: [Envelope, DirectLink]

  - term: EnvelopeItem
    definition: "A single document (PDF) attached to an Envelope. Supports multi-document envelopes."
    type: entity
    db_model: EnvelopeItem

  - term: Recipient
    definition: "A person who participates in the signing process. Has a role that determines their capabilities."
    type: entity
    db_model: Recipient
    roles:
      - SIGNER: "Must sign designated fields"
      - APPROVER: "Must approve but does not sign"
      - VIEWER: "Can view but takes no action"
      - CC: "Receives a copy when complete"
      - ASSISTANT: "Can act on behalf of another recipient"

  - term: Field
    definition: "A form element placed on a document page that a recipient must fill. 11 types available."
    type: entity
    db_model: Field
    types:
      - SIGNATURE: "Typed or drawn signature"
      - FREE_SIGNATURE: "Freehand drawing signature"
      - INITIALS: "Recipient's initials"
      - TEXT: "Free text input"
      - NUMBER: "Numeric input"
      - EMAIL: "Email address input"
      - DATE: "Date picker"
      - RADIO: "Radio button group"
      - CHECKBOX: "Checkbox"
      - DROPDOWN: "Dropdown select"
      - NAME: "Auto-filled recipient name"

  - term: Signature
    definition: "The actual signature data (typed text, drawn image, or uploaded image) associated with a signed Field."
    type: value-object
    db_model: Signature

  # Lifecycle and status
  - term: DocumentStatus
    definition: "The lifecycle state of a document."
    type: enum
    values:
      - DRAFT: "Document is being prepared, not yet sent"
      - PENDING: "Sent to recipients, awaiting signatures"
      - COMPLETED: "All recipients have completed their actions"
      - REJECTED: "A recipient has rejected the document"

  - term: SigningStatus
    definition: "The signing state of an individual recipient."
    type: enum
    values:
      - NOT_SIGNED: "Recipient has not yet acted"
      - SIGNED: "Recipient has completed signing"
      - REJECTED: "Recipient has rejected"

  - term: SigningOrder
    definition: "Controls whether recipients sign simultaneously or in sequence."
    type: enum
    values:
      - PARALLEL: "All recipients can sign at the same time"
      - SEQUENTIAL: "Recipients sign in a defined order"

  # Signing and sealing
  - term: Seal
    definition: "The cryptographic operation that embeds digital signatures into the PDF after all recipients have signed. Uses PKCS#12 (local) or Google Cloud HSM."
    type: process
    see_also: [DocumentMeta, SigningProvider]

  - term: DocumentData
    definition: "The binary PDF data stored via a swappable storage provider (database blob or S3)."
    type: entity
    db_model: DocumentData

  - term: DocumentMeta
    definition: "Signing metadata including certificate subject, issuer, timezone, redirect URL, and signing provider config."
    type: entity
    db_model: DocumentMeta

  - term: DocumentAuditLog
    definition: "Immutable log of all actions taken on a document. 18 event types covering creation, sending, signing, rejection, and completion."
    type: entity
    db_model: DocumentAuditLog

  # Access and sharing
  - term: DirectLink
    definition: "A public URL that allows anyone to sign a Template-based document without email invitation. Used for forms, waivers, and public-facing signing."
    type: entity
    db_model: DocumentDirectLink

  - term: ApiToken
    definition: "Bearer token for API authentication. Scoped to a team. Stored as SHA-256 hash."
    type: entity
    db_model: ApiToken

  # Organization
  - term: Organisation
    definition: "Top-level account entity. Contains Teams. Has its own settings, branding, and subscription."
    type: aggregate-root
    db_model: Organisation

  - term: Team
    definition: "A group within an Organisation. Documents, templates, and API tokens are scoped to a team."
    type: entity
    db_model: Team

  - term: TeamMember
    definition: "A user's membership in a team with a specific role."
    type: entity
    roles: "ADMIN | MANAGER | MEMBER"
    db_model: OrganisationMember / TeamMember

  # Infrastructure concepts
  - term: BackgroundJob
    definition: "Async task executed outside the request cycle. Three swappable providers: local DB queue, BullMQ (Redis), Inngest."
    type: entity
    db_model: BackgroundJob
    see_also: ["@RUNTIME.md#background-jobs"]

  - term: Webhook
    definition: "HTTP callback triggered by document events. Configured per team."
    type: entity
    db_model: Webhook

  - term: Subscription
    definition: "Stripe-managed billing subscription associated with an Organisation."
    type: entity
    db_model: Subscription
---

# GLOSSARY.md — Documenso

## Envelope Lifecycle

The core domain workflow expressed in ubiquitous language:

```
1. User creates an ENVELOPE (type: DOCUMENT)
2. User adds ENVELOPE ITEMS (PDF files)
3. User adds RECIPIENTS with ROLES (SIGNER, APPROVER, VIEWER, CC, ASSISTANT)
4. User places FIELDS on document pages, assigned to RECIPIENTS
5. User sends the envelope → status becomes PENDING
6. Each RECIPIENT receives email notification
7. RECIPIENTS open signing view, complete their FIELDS
8. SIGNING STATUS tracks each recipient's progress
9. When all required recipients complete → SEAL operation runs
10. SEAL embeds cryptographic signature into PDF
11. Document status becomes COMPLETED
12. AUDIT LOG records every step with timestamps and IP
```

## Signing Order

- **PARALLEL**: All recipients can sign simultaneously. No ordering constraints.
- **SEQUENTIAL**: Recipients sign in a defined order. Recipient N cannot sign until Recipient N-1 completes.

## Role Permissions

| Role | Can Sign | Can Approve | Receives Copy | Can Act for Others |
|---|---|---|---|---|
| SIGNER | Yes | No | Yes | No |
| APPROVER | No | Yes | Yes | No |
| VIEWER | No | No | Yes | No |
| CC | No | No | Yes | No |
| ASSISTANT | Yes (delegated) | No | Yes | Yes |

## Field Type Reference

See @SCHEMA.md#field-model for database details. See @UI.md#signing-flow for UI rendering.

| Type | Input Method | Validation |
|---|---|---|
| SIGNATURE | Type, draw, or upload | Required for signers |
| FREE_SIGNATURE | Freehand canvas drawing | Required for signers |
| INITIALS | Type or draw | Required for signers |
| TEXT | Free text input | Optional length limits |
| NUMBER | Numeric input | Optional min/max |
| EMAIL | Email text input | Email format validation |
| DATE | Date picker | Date format validation |
| RADIO | Radio button group | One selection required |
| CHECKBOX | Checkbox | Optional required flag |
| DROPDOWN | Select dropdown | One selection from options |
| NAME | Auto-populated | Read-only from recipient profile |
