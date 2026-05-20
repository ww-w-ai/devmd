---
devmd: errors
version: "1.0"
project: Twenty CRM

error_codes:
  authentication:
    - code: UNAUTHENTICATED
      http: 401
      description: "Missing, expired, or invalid JWT token"
      recovery: "Refresh token or redirect to login"
    - code: TOKEN_EXPIRED
      http: 401
      description: "Access token expired"
      recovery: "Use refresh token to get new access token"
    - code: INVALID_REFRESH_TOKEN
      http: 401
      description: "Refresh token invalid or revoked"
      recovery: "Full re-authentication required"

  authorization:
    - code: FORBIDDEN
      http: 403
      description: "User lacks required permission"
      recovery: "Request permission from workspace admin"
    - code: OBJECT_PERMISSION_DENIED
      http: 403
      description: "User lacks permission for this object type"
    - code: FIELD_PERMISSION_DENIED
      http: 403
      description: "User lacks permission for this field"
    - code: ROW_PERMISSION_DENIED
      http: 403
      description: "User lacks permission for this specific record (row-level security)"

  validation:
    - code: VALIDATION_ERROR
      http: 400
      description: "Input data fails validation"
      payload: "{ field: string, message: string, constraint: string }"
    - code: SCHEMA_VERSION_MISMATCH
      http: 409
      description: "Client schema version differs from server. Metadata has changed."
      recovery: "Refetch schema and retry"
    - code: DUPLICATE_ENTRY
      http: 409
      description: "Unique constraint violation"
    - code: RELATION_CONSTRAINT
      http: 400
      description: "Foreign key or relation constraint violation"

  not_found:
    - code: NOT_FOUND
      http: 404
      description: "Requested resource does not exist or has been soft-deleted"
    - code: WORKSPACE_NOT_FOUND
      http: 404
      description: "Workspace ID in token does not match any workspace"

  server:
    - code: INTERNAL_SERVER_ERROR
      http: 500
      description: "Unexpected server error"
      tracking: "Sentry event ID returned in response"
    - code: SERVICE_UNAVAILABLE
      http: 503
      description: "Downstream service (DB, Redis, ClickHouse) unreachable"

  rate_limit:
    - code: TOO_MANY_REQUESTS
      http: 429
      description: "Rate limit exceeded"
      headers: "Retry-After (seconds)"

exception_hierarchy:
  base: BaseGraphQLError
  children:
    - AuthenticationError
    - ForbiddenError
    - NotFoundError
    - ValidationError
    - ConflictError
    - InternalServerError
    - ServiceUnavailableError

retry_policy:
  client_side:
    - error: TOKEN_EXPIRED
      action: "Refresh token, retry original request (max 1 retry)"
    - error: SCHEMA_VERSION_MISMATCH
      action: "Refetch schema, retry original request (max 1 retry)"
    - error: SERVICE_UNAVAILABLE
      action: "Exponential backoff (1s, 2s, 4s), max 3 retries"
    - error: TOO_MANY_REQUESTS
      action: "Wait for Retry-After header value, then retry"
  server_side:
    - context: webhook_dispatch
      action: "3 retries with exponential backoff (1s, 5s, 30s)"
    - context: email_sync
      action: "Retry on next sync cycle (5 min interval)"
    - context: worker_job
      action: "BullMQ automatic retry with backoff (configurable per queue)"

metrics:
  - name: graphql_operation_401_total
    type: counter
    description: "Total 401 errors per operation"
  - name: graphql_operation_500_total
    type: counter
    description: "Total 500 errors per operation"
  - name: graphql_operation_error_total
    type: counter
    labels: [operation_name, error_code]
    description: "Total errors by operation and error code"
---

# Twenty CRM Error Handling

## GraphQL Error Format

All GraphQL errors follow a consistent format via BaseGraphQLError. See @API.md#error-format.

```json
{
  "errors": [
    {
      "message": "You do not have permission to access Person records",
      "locations": [{ "line": 2, "column": 3 }],
      "path": ["findPeople"],
      "extensions": {
        "code": "OBJECT_PERMISSION_DENIED",
        "status": 403,
        "objectName": "person",
        "requiredPermission": "READ_PERSON"
      }
    }
  ]
}
```

## Exception Handler Module

The backend exception handler module catches all unhandled exceptions and transforms them into proper GraphQL errors.

**Processing pipeline:**
1. Exception thrown in resolver/service
2. NestJS exception filter catches it
3. Maps to appropriate BaseGraphQLError subclass
4. Adds `extensions` with error code, status, and context
5. Logs to application log with correlation ID. See @LOGGING.md
6. Reports to Sentry if severity >= ERROR
7. Returns formatted GraphQL error response

## Frontend Error Handling

### Apollo Error Link

```typescript
// Error handling chain in Apollo Client
const errorLink = onError(({ graphQLErrors, networkError, operation }) => {
  if (graphQLErrors) {
    for (const error of graphQLErrors) {
      switch (error.extensions?.code) {
        case 'UNAUTHENTICATED':
        case 'TOKEN_EXPIRED':
          // Trigger token refresh + retry
          return refreshTokenAndRetry(operation);
        case 'SCHEMA_VERSION_MISMATCH':
          // Refetch schema + retry
          return refetchSchemaAndRetry(operation);
        case 'FORBIDDEN':
          // Show permission error toast
          showErrorToast('Permission denied');
          break;
        default:
          // Log to Sentry, show generic error
          Sentry.captureException(error);
          showErrorToast(error.message);
      }
    }
  }
  if (networkError) {
    // Network-level error (no response from server)
    showErrorToast('Network error. Please check your connection.');
  }
});
```

### Error Handler Module (Frontend)

The `error-handler` module in twenty-front provides:
- **useGraphQLErrorHandler** hook — attaches to Apollo error link
- **ErrorBoundary** component — catches React render errors
- **useSnackBar** — displays error toasts to user
- **Sentry integration** — captures errors with user context, workspace ID, operation name

### Token Renewal Flow

```
Client sends request with expired access token
  → Server returns 401 UNAUTHENTICATED
  → Apollo error link intercepts
  → Sends refresh token to /auth/refresh
  → Receives new access + refresh tokens
  → Retries original request with new access token
  → If refresh also fails → redirect to login
```

## Sentry Integration

Both frontend and backend use Sentry 10 for error tracking.

**Backend configuration:**
- DSN configured via `SENTRY_DSN` environment variable
- Release tracking via git commit SHA
- Environment tagging (development, staging, production)
- User context: workspace ID, user ID, user email
- Transaction tracing for GraphQL operations

**Frontend configuration:**
- DSN configured via `REACT_APP_SENTRY_DSN`
- Browser session replay for error reproduction
- Breadcrumbs: Apollo operations, route changes, user clicks
- Performance monitoring: page load, GraphQL operation duration

## Metrics

Error metrics emitted via OpenTelemetry. See @LOGGING.md#observability.

| Metric | Type | Labels | Purpose |
|---|---|---|---|
| `graphql_operation_401_total` | Counter | operation_name | Auth failure rate |
| `graphql_operation_500_total` | Counter | operation_name | Server error rate |
| `graphql_operation_error_total` | Counter | operation_name, error_code | Error breakdown |
| `webhook_delivery_failure_total` | Counter | webhook_id, status_code | Webhook reliability |
| `worker_job_failure_total` | Counter | queue_name, error_type | Worker reliability |

## Cross-References

- @API.md — Error format in API responses
- @LOGGING.md — Structured error logging and tracing
- @SECURITY.md — Authentication errors and token management
- @RUNTIME.md — Worker job retry policies
