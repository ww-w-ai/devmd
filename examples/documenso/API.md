---
devmd: api
version: "1.0"
project: documenso
updated: 2026-05-13

protocol: trpc
trpc_version: 11
openapi: true
openapi_version: "3.1"

base_urls:
  internal_trpc: "/trpc"
  v2_openapi: "/api/v2"
  v1_rest: "/api/v1"  # deprecated

auth_methods:
  - type: bearer
    header: "Authorization: Bearer {apiToken}"
    scope: "v2 OpenAPI endpoints"
  - type: session
    mechanism: "encrypted session cookie"
    scope: "internal tRPC (browser)"

versioning:
  strategy: "URL path prefix"
  current: v2
  deprecated: [v1]

content_type: "application/json"
error_format: "@ERRORS.md#app-error"

rate_limiting:
  backend: database
  strategy: "sliding window per IP + user"
  see: "@SECURITY.md#rate-limiting"

procedure_types:
  - name: base
    auth: none
    description: "No authentication required"
  - name: maybeAuthenticated
    auth: optional
    description: "Works with or without auth, behavior varies"
  - name: authenticated
    auth: required
    description: "Requires valid session or API token"
  - name: admin
    auth: required
    role: ADMIN
    description: "Requires admin role"
---

# API.md — Documenso

## API Architecture

Documenso uses a **dual-layer API**:

1. **Internal tRPC** (`/trpc`) — Used by the React frontend. Full type safety via tRPC client. Session cookie auth.
2. **v2 OpenAPI** (`/api/v2`) — Public REST API auto-generated from tRPC routers via `trpc-to-openapi`. Bearer token auth.

Both layers share the same tRPC router definitions in `@documenso/trpc`.

```
Browser ──cookie──► /trpc ──────► tRPC Router ──► @documenso/lib
                                       │
External ──bearer──► /api/v2 ──► OpenAPI Bridge ──┘
```

## tRPC Routers (15)

### document

Operations on document-type envelopes.

| Procedure | Type | Description |
|---|---|---|
| `document.findDocuments` | authenticated | List documents with filters, pagination, sorting |
| `document.findDocumentById` | authenticated | Get document by ID with relations |
| `document.createDocument` | authenticated | Create a new document envelope |
| `document.deleteDocument` | authenticated | Soft-delete a document |
| `document.sendDocument` | authenticated | Transition DRAFT → PENDING, notify recipients |
| `document.resendDocument` | authenticated | Re-send notifications to unsigned recipients |
| `document.duplicateDocument` | authenticated | Clone a document as new DRAFT |
| `document.moveDocumentToTeam` | authenticated | Transfer document to another team |
| `document.downloadDocument` | authenticated | Get signed PDF |
| `document.downloadAuditLog` | authenticated | Get audit log PDF |
| `document.searchDocuments` | authenticated | Full-text search across documents |

### template

| Procedure | Type | Description |
|---|---|---|
| `template.findTemplates` | authenticated | List templates |
| `template.findTemplateById` | authenticated | Get template by ID |
| `template.createTemplate` | authenticated | Create template envelope |
| `template.updateTemplate` | authenticated | Update template settings |
| `template.deleteTemplate` | authenticated | Delete template |
| `template.duplicateTemplate` | authenticated | Clone template |
| `template.createDocumentFromTemplate` | authenticated | Instantiate document from template |

### envelope

| Procedure | Type | Description |
|---|---|---|
| `envelope.findEnvelopes` | authenticated | Unified query across documents and templates |
| `envelope.getEnvelope` | authenticated | Get envelope with all relations |
| `envelope.updateEnvelope` | authenticated | Update envelope metadata |

### recipient

| Procedure | Type | Description |
|---|---|---|
| `recipient.addRecipients` | authenticated | Add recipients to envelope |
| `recipient.updateRecipient` | authenticated | Update recipient details/role |
| `recipient.removeRecipient` | authenticated | Remove recipient from envelope |
| `recipient.getRecipient` | maybeAuthenticated | Get recipient by token (signing view) |
| `recipient.completeRecipient` | base | Mark recipient as completed |
| `recipient.rejectDocument` | base | Recipient rejects the document |

### field

| Procedure | Type | Description |
|---|---|---|
| `field.addFields` | authenticated | Add fields to envelope pages |
| `field.updateField` | authenticated | Update field position/type/config |
| `field.removeField` | authenticated | Remove field |
| `field.signField` | base | Insert signature/value into field |
| `field.removeSignature` | base | Remove inserted value from field |

### folder

| Procedure | Type | Description |
|---|---|---|
| `folder.findFolders` | authenticated | List folders |
| `folder.createFolder` | authenticated | Create folder |
| `folder.updateFolder` | authenticated | Rename/move folder |
| `folder.deleteFolder` | authenticated | Delete folder |
| `folder.moveToFolder` | authenticated | Move document/template to folder |

### auth

| Procedure | Type | Description |
|---|---|---|
| `auth.signup` | base | Register new account |
| `auth.signIn` | base | Email/password login |
| `auth.signOut` | authenticated | Destroy session |
| `auth.verifyEmail` | base | Verify email token |
| `auth.forgotPassword` | base | Request password reset |
| `auth.resetPassword` | base | Reset password with token |

### profile

| Procedure | Type | Description |
|---|---|---|
| `profile.getProfile` | authenticated | Get current user profile |
| `profile.updateProfile` | authenticated | Update name, signature |
| `profile.updatePassword` | authenticated | Change password |
| `profile.deleteAccount` | authenticated | Delete user account |

### team

| Procedure | Type | Description |
|---|---|---|
| `team.findTeams` | authenticated | List teams in org |
| `team.createTeam` | authenticated | Create team |
| `team.updateTeam` | authenticated | Update team settings |
| `team.deleteTeam` | authenticated | Delete team |
| `team.findTeamMembers` | authenticated | List team members |
| `team.addTeamMember` | authenticated | Invite member to team |
| `team.removeTeamMember` | authenticated | Remove member from team |
| `team.updateTeamMember` | authenticated | Change member role |

### organisation

| Procedure | Type | Description |
|---|---|---|
| `organisation.getOrganisation` | authenticated | Get org details |
| `organisation.updateOrganisation` | authenticated | Update org settings |
| `organisation.findMembers` | authenticated | List org members |
| `organisation.inviteMember` | authenticated | Invite to org |
| `organisation.removeMember` | authenticated | Remove from org |

### api-token

| Procedure | Type | Description |
|---|---|---|
| `apiToken.findTokens` | authenticated | List API tokens |
| `apiToken.createToken` | authenticated | Generate new API token |
| `apiToken.deleteToken` | authenticated | Revoke API token |

### webhook

| Procedure | Type | Description |
|---|---|---|
| `webhook.findWebhooks` | authenticated | List webhooks |
| `webhook.createWebhook` | authenticated | Register webhook URL |
| `webhook.updateWebhook` | authenticated | Update webhook config |
| `webhook.deleteWebhook` | authenticated | Remove webhook |

### admin

| Procedure | Type | Description |
|---|---|---|
| `admin.findUsers` | admin | List all users |
| `admin.findDocuments` | admin | List all documents |
| `admin.updateUser` | admin | Modify user |
| `admin.deleteUser` | admin | Delete user |
| `admin.getStats` | admin | Platform statistics |

### enterprise

| Procedure | Type | Description |
|---|---|---|
| `enterprise.getEmbeddingConfig` | authenticated | Get embedding settings |
| `enterprise.updateEmbeddingConfig` | authenticated | Update embedding config |

### embedding

| Procedure | Type | Description |
|---|---|---|
| `embedding.createEmbeddingToken` | authenticated | Generate embed signing token |
| `embedding.getEmbeddingDocument` | base | Get document for embedded view |

## OpenAPI Mapping

tRPC procedures are exposed as REST endpoints via `trpc-to-openapi`:

```
tRPC: document.findDocuments
REST: GET /api/v2/document

tRPC: document.createDocument
REST: POST /api/v2/document

tRPC: document.findDocumentById
REST: GET /api/v2/document/{id}

tRPC: document.sendDocument
REST: POST /api/v2/document/{id}/send

tRPC: recipient.addRecipients
REST: POST /api/v2/document/{id}/recipient

tRPC: field.addFields
REST: POST /api/v2/document/{id}/field
```

## Request/Response Format

### Successful Response

```json
{
  "result": {
    "data": { ... }
  }
}
```

### Error Response

See @ERRORS.md for full error code reference.

```json
{
  "error": {
    "message": "Document not found",
    "code": "NOT_FOUND",
    "data": {
      "code": "NOT_FOUND",
      "httpStatus": 404,
      "path": "document.findDocumentById"
    }
  }
}
```

### Pagination

```typescript
// Request
{
  page: number,      // 1-indexed
  perPage: number,   // default 10, max 100
  orderBy?: {
    column: string,
    direction: 'asc' | 'desc'
  }
}

// Response
{
  data: T[],
  totalPages: number,
  currentPage: number,
  perPage: number
}
```

## Authentication Headers

### API Token (v2 OpenAPI)

```
Authorization: Bearer dt_xxxxxxxxxxxxxxxxxxxx
```

Token format: `dt_` prefix + 40 character random string. Stored as SHA-256 hash in database. See @SCHEMA.md#apitoken.

### Session Cookie (Internal tRPC)

```
Cookie: __session=encrypted_session_token
```

Session is encrypted server-side. Contains userId and expiry. See @SECURITY.md#authentication.

## v1 REST API (Deprecated)

The v1 API built with `ts-rest` is deprecated. All new integrations should use v2. v1 endpoints are still served at `/api/v1` but receive no new features.

## Cross-References

- Error codes: @ERRORS.md
- Authentication: @SECURITY.md#authentication
- Database models: @SCHEMA.md
- tRPC router source: @ARCHITECTURE.md#domain-module-organization
