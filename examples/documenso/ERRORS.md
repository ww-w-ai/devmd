---
devmd: errors
version: "1.0"
project: documenso
updated: 2026-05-13

error_class: AppError
error_module: "@documenso/lib/errors/app-error"
total_codes: 22

patterns:
  - "AppError wraps all domain errors"
  - "Internal message (developer) vs user message (end user) separation"
  - "Each AppErrorCode maps to HTTP status + tRPC error code"
  - "Errors are caught at tRPC middleware layer and formatted consistently"
  - "Unexpected errors are logged with Pino, sanitized before sending to client"

retry_policy:
  default: "No automatic retry on API errors"
  background_jobs: "Up to 3 retries with exponential backoff — see @RUNTIME.md#retry-policy"
  webhooks: "Up to 3 retries with exponential backoff"
  email: "Up to 3 retries via background job system"
---

# ERRORS.md — Documenso

## AppError Class

All domain errors extend `AppError`. This is the only error type that should be thrown from `@documenso/lib`.

```typescript
class AppError extends Error {
  public readonly code: AppErrorCode;
  public readonly httpStatus: number;
  public readonly trpcCode: TRPCErrorCode;
  public readonly userMessage: string;

  constructor(
    code: AppErrorCode,
    message?: string,        // Internal message (logged, not shown to user)
    userMessage?: string     // User-facing message (shown in UI)
  ) { ... }
}
```

### Usage Pattern

```typescript
// In domain logic (@documenso/lib)
throw new AppError(
  'NOT_FOUND',
  `Envelope ${id} not found in team ${teamId}`,  // internal (logged)
  'Document not found'                              // user-facing
);

// In tRPC middleware (automatic)
// AppError → TRPCError with correct code + HTTP status
// Non-AppError → INTERNAL_SERVER_ERROR (500), generic user message
```

## Error Code Reference

| AppErrorCode | HTTP Status | tRPC Code | Description |
|---|---|---|---|
| `ALREADY_EXISTS` | 409 | CONFLICT | Resource already exists (duplicate email, team URL, etc.) |
| `EXPIRED_CODE` | 400 | BAD_REQUEST | Verification/reset code has expired |
| `INVALID_BODY` | 400 | BAD_REQUEST | Request body fails Zod validation |
| `INVALID_REQUEST` | 400 | BAD_REQUEST | General invalid request (missing params, bad state) |
| `NOT_FOUND` | 404 | NOT_FOUND | Resource does not exist or is not accessible |
| `UNAUTHORIZED` | 401 | UNAUTHORIZED | No valid session or API token |
| `FORBIDDEN` | 403 | FORBIDDEN | Authenticated but insufficient permissions |
| `TOO_MANY_REQUESTS` | 429 | TOO_MANY_REQUESTS | Rate limit exceeded |
| `INTERNAL_SERVER_ERROR` | 500 | INTERNAL_SERVER_ERROR | Unexpected server error |
| `BAD_REQUEST` | 400 | BAD_REQUEST | General bad request |
| `CONFLICT` | 409 | CONFLICT | State conflict (document already sent, etc.) |
| `GONE` | 410 | NOT_FOUND | Resource has been permanently removed |
| `UNPROCESSABLE_ENTITY` | 422 | BAD_REQUEST | Semantically invalid (valid JSON but bad data) |
| `NOT_IMPLEMENTED` | 501 | INTERNAL_SERVER_ERROR | Feature not yet implemented |
| `SERVICE_UNAVAILABLE` | 503 | INTERNAL_SERVER_ERROR | External service down (signing, email, etc.) |
| `DOCUMENT_SEND_FAILED` | 500 | INTERNAL_SERVER_ERROR | Failed to transition document to PENDING |
| `DOCUMENT_SEAL_FAILED` | 500 | INTERNAL_SERVER_ERROR | Cryptographic sealing failed |
| `SIGNING_FAILED` | 500 | INTERNAL_SERVER_ERROR | PDF signing operation failed |
| `EMAIL_SEND_FAILED` | 500 | INTERNAL_SERVER_ERROR | Email delivery failed |
| `STORAGE_ERROR` | 500 | INTERNAL_SERVER_ERROR | File storage read/write failed |
| `PROFILE_URL_TAKEN` | 409 | CONFLICT | Profile URL already in use |
| `TEAM_URL_TAKEN` | 409 | CONFLICT | Team URL already in use |

## Error Hierarchy by Domain

### Document Lifecycle Errors

```
DRAFT → PENDING:
  - INVALID_REQUEST: "Missing recipients" | "No fields assigned" | "No document items"
  - DOCUMENT_SEND_FAILED: Envelope send operation failed
  - EMAIL_SEND_FAILED: Recipient notification failed (non-blocking, retried)

PENDING → COMPLETED:
  - DOCUMENT_SEAL_FAILED: PDF sealing failed (cryptographic error)
  - SIGNING_FAILED: Underlying signing provider error
  - STORAGE_ERROR: Cannot read/write document data

PENDING → REJECTED:
  - (No error — rejection is a valid terminal state)
```

### Authentication Errors

```
Login:
  - UNAUTHORIZED: Invalid credentials
  - EXPIRED_CODE: 2FA code expired
  - TOO_MANY_REQUESTS: Too many login attempts

Session:
  - UNAUTHORIZED: Session expired or invalid
  - FORBIDDEN: Valid session but wrong role/team

API Token:
  - UNAUTHORIZED: Invalid or expired token
  - FORBIDDEN: Token lacks permission for resource
```

### Organization/Team Errors

```
  - ALREADY_EXISTS: Team URL or org URL taken
  - TEAM_URL_TAKEN: Specific team URL conflict
  - PROFILE_URL_TAKEN: Specific profile URL conflict
  - FORBIDDEN: Not an admin/manager of the team
  - NOT_FOUND: Team or org does not exist
```

## Client-Side Error Handling

```typescript
// React component pattern
import { TRPCClientError } from '@trpc/client';

try {
  await trpc.document.sendDocument.mutate({ documentId });
} catch (error) {
  if (error instanceof TRPCClientError) {
    // error.data.code → AppErrorCode
    // error.message → user-facing message
    toast.error(error.message);
  }
}
```

## Validation Errors (Zod)

Input validation uses Zod schemas defined alongside tRPC procedures. Validation failures are automatically caught by tRPC and returned as `INVALID_BODY`:

```json
{
  "error": {
    "message": "Input validation failed",
    "code": "BAD_REQUEST",
    "data": {
      "code": "INVALID_BODY",
      "zodError": {
        "issues": [
          {
            "path": ["email"],
            "message": "Invalid email format",
            "code": "invalid_string"
          }
        ]
      }
    }
  }
}
```

## Retry Policy

### API Requests
No automatic retry. Clients should implement their own retry logic for `429` and `5xx` responses.

### Background Jobs
See @RUNTIME.md#retry-policy for background job retry behavior.

### Webhooks
Failed webhook deliveries are retried up to 3 times with exponential backoff (1min, 5min, 30min).

## Cross-References

- API response format: @API.md#request-response-format
- Logging of errors: @LOGGING.md#error-logging
- Background job retries: @RUNTIME.md#retry-policy
- Rate limiting: @SECURITY.md#rate-limiting
