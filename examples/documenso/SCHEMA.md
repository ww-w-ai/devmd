---
devmd: schema
version: "1.0"
project: documenso
updated: 2026-05-13

database: postgresql
version: "15"
orm: "Prisma 6 + Kysely"
total_models: 45
total_enums: 26
migration_tool: prisma-migrate

naming_conventions:
  tables: PascalCase
  columns: camelCase
  enums: UPPER_SNAKE_CASE
  foreign_keys: "{relationName}Id"
  indexes: "named explicitly in Prisma schema"

key_patterns:
  - "Polymorphic Envelope (DOCUMENT | TEMPLATE) as aggregate root"
  - "Soft references via nullable FKs for optional relations"
  - "JSON columns for flexible metadata (fieldMeta, authOptions)"
  - "DateTime tracking (createdAt, updatedAt, deletedAt) on most models"
  - "UUID primary keys on newer models, autoincrement Int on legacy models"
---

# SCHEMA.md — Documenso

## Entity Relationship Overview

```
Organisation (1) ──── (*) Team
     │                      │
     │                      ├── (*) Envelope
     │                      │        ├── (*) EnvelopeItem
     │                      │        ├── (*) Recipient ── (*) Field ── (0..1) Signature
     │                      │        └── (1) DocumentData
     │                      │
     │                      ├── (*) ApiToken
     │                      ├── (*) Webhook
     │                      └── (*) Folder
     │
     ├── (*) OrganisationMember ── (1) User
     ├── (0..1) Subscription
     └── (*) OrganisationGlobalSettings
```

## Core Models

### Envelope (Aggregate Root)

The central entity. Polymorphic via `envelopeType`.

```prisma
model Envelope {
  id              Int              @id @default(autoincrement())
  envelopeType    EnvelopeType     @default(DOCUMENT)
  status          DocumentStatus   @default(DRAFT)
  title           String
  visibility      EnvelopeVisibility @default(PRIVATE)
  signingOrder    DocumentSigningOrder @default(PARALLEL)

  // Ownership
  userId          Int
  teamId          Int
  organisationId  String

  // Relations
  user            User             @relation(...)
  team            Team             @relation(...)
  organisation    Organisation     @relation(...)
  items           EnvelopeItem[]
  recipients      Recipient[]
  documentData    DocumentData?
  documentMeta    DocumentMeta?
  auditLogs       DocumentAuditLog[]
  directLink      DocumentDirectLink?
  folder          Folder?          @relation(...)

  // Template-specific
  templateDirectLink  TemplateDirectLink?

  // Timestamps
  createdAt       DateTime         @default(now())
  updatedAt       DateTime         @updatedAt
  completedAt     DateTime?
  deletedAt       DateTime?

  @@index([teamId, status])
  @@index([userId])
  @@index([organisationId])
}
```

### EnvelopeItem

```prisma
model EnvelopeItem {
  id            Int       @id @default(autoincrement())
  envelopeId    Int
  documentDataId String
  name          String?
  order         Int       @default(0)

  envelope      Envelope     @relation(...)
  documentData  DocumentData @relation(...)

  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
}
```

### Recipient

```prisma
model Recipient {
  id              Int              @id @default(autoincrement())
  envelopeId      Int
  email           String
  name            String           @default("")
  role            RecipientRole    @default(SIGNER)
  signingStatus   SigningStatus    @default(NOT_SIGNED)
  signingOrder    Int?
  token           String           @unique @default(uuid())
  authOptions     Json?            // Access auth requirements

  envelope        Envelope         @relation(...)
  fields          Field[]
  signatures      Signature[]

  readStatus      ReadStatus       @default(NOT_OPENED)
  sendStatus      SendStatus       @default(NOT_SENT)

  signedAt        DateTime?
  createdAt       DateTime         @default(now())
  updatedAt       DateTime         @updatedAt

  @@index([envelopeId])
  @@index([token])
}
```

### Field

```prisma
model Field {
  id              Int          @id @default(autoincrement())
  recipientId     Int
  envelopeId      Int
  type            FieldType
  page            Int
  positionX       Decimal
  positionY       Decimal
  width           Decimal
  height          Decimal
  customText      String       @default("")
  inserted        Boolean      @default(false)
  fieldMeta       Json?        // Type-specific config (options, validation, etc.)

  recipient       Recipient    @relation(...)
  envelope        Envelope     @relation(...)
  signature       Signature?

  createdAt       DateTime     @default(now())
  updatedAt       DateTime     @updatedAt

  @@index([envelopeId])
  @@index([recipientId])
}
```

### Signature

```prisma
model Signature {
  id                    Int      @id @default(autoincrement())
  fieldId               Int      @unique
  recipientId           Int
  signatureImageAsBase64 String?
  typedSignature        String?

  field                 Field    @relation(...)
  recipient             Recipient @relation(...)

  createdAt             DateTime @default(now())
  updatedAt             DateTime @updatedAt
}
```

### DocumentData

```prisma
model DocumentData {
  id     String              @id @default(cuid())
  type   DocumentDataType    // S3_PATH | BYTES_64
  data   String              // S3 key or base64-encoded PDF
}
```

### DocumentMeta

```prisma
model DocumentMeta {
  id                    String   @id @default(cuid())
  envelopeId            Int      @unique
  subject               String?
  message               String?
  timezone              String   @default("Etc/UTC")
  dateFormat            String   @default("yyyy-MM-dd hh:mm a")
  redirectUrl           String?
  signingCertificate    String?  // Custom P12 certificate
  language              String?
  typedSignatureEnabled Boolean  @default(true)
  distributionMethod    DocumentDistributionMethod @default(EMAIL)

  envelope              Envelope @relation(...)
}
```

### DocumentAuditLog

```prisma
model DocumentAuditLog {
  id            String   @id @default(cuid())
  envelopeId    Int
  type          DocumentAuditLogType
  data          Json     // Event-specific payload
  name          String?
  email         String?
  userId        Int?
  ipAddress     String?
  userAgent     String?

  envelope      Envelope @relation(...)

  createdAt     DateTime @default(now())

  @@index([envelopeId])
}
```

## User & Organization Models

### User

```prisma
model User {
  id                Int               @id @default(autoincrement())
  name              String?
  email             String            @unique
  emailVerified     DateTime?
  password          String?           // bcrypt hash
  source            String?
  signature         String?           // Default signature image
  roles             Role[]            @default([USER])
  identityProvider  IdentityProvider  @default(DOCUMENSO)
  twoFactorEnabled  Boolean          @default(false)
  twoFactorSecret   String?

  // Relations
  envelopes         Envelope[]
  accounts          Account[]         // OAuth accounts
  sessions          Session[]
  passkeys          Passkey[]
  organisationMembers OrganisationMember[]
  apiTokens         ApiToken[]

  createdAt         DateTime          @default(now())
  updatedAt         DateTime          @updatedAt
  lastSignedIn      DateTime          @default(now())

  @@index([email])
}
```

### Organisation

```prisma
model Organisation {
  id              String    @id @default(cuid())
  name            String
  url             String    @unique  // slug
  customerId      String?   // Stripe customer ID
  avatarImageId   String?

  members         OrganisationMember[]
  teams           Team[]
  subscription    Subscription?
  globalSettings  OrganisationGlobalSettings?
  claims          OrganisationClaim[]

  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@index([url])
}
```

### Team

```prisma
model Team {
  id              Int       @id @default(autoincrement())
  name            String
  url             String    // slug within org
  organisationId  String
  avatarImageId   String?

  organisation    Organisation @relation(...)
  members         TeamMember[]
  envelopes       Envelope[]
  apiTokens       ApiToken[]
  webhooks        Webhook[]

  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@unique([organisationId, url])
}
```

## Auth Models

### Session

```prisma
model Session {
  id            String   @id @default(cuid())
  userId        Int
  token         String   @unique
  expiresAt     DateTime

  user          User     @relation(...)

  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
}
```

### Account (OAuth)

```prisma
model Account {
  id                String  @id @default(cuid())
  userId            Int
  type              String
  provider          String
  providerAccountId String

  // OAuth tokens
  access_token      String?
  refresh_token     String?
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String?

  user              User    @relation(...)

  @@unique([provider, providerAccountId])
}
```

### Passkey

```prisma
model Passkey {
  id                  String    @id @default(cuid())
  userId              Int
  name                String
  credentialId        Bytes     @unique
  credentialPublicKey Bytes
  counter             BigInt
  credentialDeviceType String
  credentialBackedUp  Boolean
  transports          String[]

  user                User      @relation(...)

  createdAt           DateTime  @default(now())
  updatedAt           DateTime  @updatedAt
}
```

## Infrastructure Models

### BackgroundJob

```prisma
model BackgroundJob {
  id            String              @id @default(cuid())
  status        BackgroundJobStatus @default(PENDING)
  name          String
  version       String              @default("1")
  payload       Json
  retryCount    Int                 @default(0)
  maxRetries    Int                 @default(3)
  lastRetriedAt DateTime?

  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt
  completedAt   DateTime?

  @@index([status, name])
}
```

### Webhook

```prisma
model Webhook {
  id            String   @id @default(cuid())
  teamId        Int
  url           String
  eventTriggers String[] // Array of event type strings
  secret        String?  // HMAC signing secret
  enabled       Boolean  @default(true)

  team          Team     @relation(...)

  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
}
```

### ApiToken

```prisma
model ApiToken {
  id            Int      @id @default(autoincrement())
  name          String
  token         String   @unique  // SHA-256 hash of actual token
  userId        Int
  teamId        Int
  expiresAt     DateTime?

  user          User     @relation(...)
  team          Team     @relation(...)

  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
}
```

## All Enums

```prisma
enum EnvelopeType          { DOCUMENT, TEMPLATE }
enum DocumentStatus        { DRAFT, PENDING, COMPLETED, REJECTED }
enum RecipientRole         { CC, SIGNER, VIEWER, APPROVER, ASSISTANT }
enum SigningStatus          { NOT_SIGNED, SIGNED, REJECTED }
enum SendStatus            { NOT_SENT, SENT }
enum ReadStatus            { NOT_OPENED, OPENED }
enum FieldType             { SIGNATURE, FREE_SIGNATURE, INITIALS, TEXT, NUMBER, EMAIL, DATE, RADIO, CHECKBOX, DROPDOWN, NAME }
enum DocumentSigningOrder  { PARALLEL, SEQUENTIAL }
enum DocumentDataType      { S3_PATH, BYTES_64 }
enum DocumentDistributionMethod { EMAIL, NONE }
enum BackgroundJobStatus   { PENDING, PROCESSING, COMPLETED, FAILED }
enum Role                  { USER, ADMIN }
enum IdentityProvider      { DOCUMENSO, GOOGLE, OIDC }
enum EnvelopeVisibility    { PRIVATE, EVERYONE }
enum OrganisationMemberRole { ADMIN, MANAGER, MEMBER }
enum TeamMemberRole        { ADMIN, MANAGER, MEMBER }

// Audit log types (18 events)
enum DocumentAuditLogType {
  DOCUMENT_CREATED
  DOCUMENT_OPENED
  DOCUMENT_SENT
  DOCUMENT_COMPLETED
  DOCUMENT_DELETED
  DOCUMENT_FIELD_INSERTED
  DOCUMENT_FIELD_UNINSERTED
  DOCUMENT_META_UPDATED
  DOCUMENT_RECIPIENT_COMPLETED
  DOCUMENT_RECIPIENT_REJECTED
  DOCUMENT_RECIPIENT_ADDED
  DOCUMENT_RECIPIENT_REMOVED
  EMAIL_SENT
  DOCUMENT_MOVED_TO_TEAM
  DOCUMENT_GLOBAL_AUTH_ACCESS_UPDATED
  DOCUMENT_GLOBAL_AUTH_ACTION_UPDATED
  DOCUMENT_RECIPIENT_AUTH_ACCESS_UPDATED
  DOCUMENT_RECIPIENT_AUTH_ACTION_UPDATED
}
```

## Migration Rules

1. All migrations run via `prisma migrate dev` (development) or `prisma migrate deploy` (production).
2. Migration files are committed to `packages/prisma/migrations/`.
3. **Never** run `prisma migrate reset` on production.
4. Destructive changes (column drops, table drops) require a two-phase migration: deprecate first, remove in next release.
5. New models must include `createdAt` and `updatedAt` timestamps.
6. All foreign keys must have explicit `onDelete` behavior defined.
7. Indexes must be named and added for all frequently queried foreign keys.

## Cross-References

- Model relationships: @GLOSSARY.md#envelope-lifecycle
- API exposure: @API.md#trpc-routers
- Audit log details: @LOGGING.md#audit-events
- Background job lifecycle: @RUNTIME.md#background-jobs
