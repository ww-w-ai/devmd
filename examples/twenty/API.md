---
devmd: api
version: "1.0"
project: Twenty CRM
protocol: graphql
secondary_protocol: rest
style: three-tier

endpoints:
  core:
    path: /api
    schema: dynamic
    description: "Workspace data CRUD — generated from metadata"
    auth: "JWT (access token) + workspace context"
    pagination: cursor-based
    features: [filtering, sorting, aggregation, connection-types]
  metadata:
    path: /metadata
    schema: static
    description: "Schema management — create/update/delete objects, fields, relations"
    auth: "JWT + DATA_MODEL permission"
  admin:
    path: /admin
    schema: static
    description: "Admin panel — workspace management, billing, feature flags"
    auth: "JWT + workspace owner/admin role"
  rest:
    path: /api/v1
    schema: auto-generated
    description: "Dynamic REST API mirroring Core GraphQL. OpenAPI spec auto-generated."
    auth: "Bearer token or API key"
  webhooks:
    description: "Outbound HTTP callbacks on data mutations"
    trigger: database_change
    payload_format: json

graphql_conventions:
  naming:
    query_single: "findOne{Object}"
    query_many: "find{Object}s"
    mutation_create: "createOne{Object}"
    mutation_create_many: "create{Object}s"
    mutation_update: "updateOne{Object}"
    mutation_update_many: "update{Object}s"
    mutation_delete: "deleteOne{Object}"
    mutation_delete_many: "delete{Object}s"
    mutation_restore: "restoreOne{Object}"
  pagination:
    style: cursor
    args: [first, last, before, after]
    response: "{Object}Connection { edges { node cursor } pageInfo { hasNextPage hasPreviousPage startCursor endCursor } totalCount }"
  filtering:
    style: nested-object
    operators: [eq, neq, gt, gte, lt, lte, in, is, like, ilike, startsWith, contains]
    logical: [and, or, not]
    example: "filter: { name: { firstName: { like: \"%John%\" } }, company: { name: { eq: \"Acme\" } } }"
  sorting:
    style: array
    format: "orderBy: [{ name: { firstName: AscNullsLast } }]"
    directions: [AscNullsFirst, AscNullsLast, DescNullsFirst, DescNullsLast]

error_format:
  type: GraphQLError
  extensions:
    code: "string (UNAUTHENTICATED, FORBIDDEN, NOT_FOUND, VALIDATION_ERROR, INTERNAL_SERVER_ERROR)"
    status: "number (HTTP status equivalent)"
    message: "string (human-readable)"

headers:
  request:
    - name: Authorization
      format: "Bearer {access_token}"
      required: true
    - name: X-Schema-Version
      format: "string (schema version hash)"
      required: false
      purpose: Optimistic concurrency — reject requests with stale schema
    - name: x-locale
      format: "string (e.g., en, fr, ko)"
      required: false
      purpose: i18n for field labels and error messages
  response:
    - name: X-Schema-Version
      purpose: Current schema version for client cache invalidation

rate_limits:
  default: "100 req/s per workspace"
  burst: "200 req/s"
  api_key: "50 req/s"
---

# Twenty CRM API

## Three-Tier GraphQL Architecture

Three separate GraphQL endpoints serve different purposes. See @ARCHITECTURE.md#adrs for the ADR.

### Core API (`/api`)

The primary data API. Schema is **dynamically generated** from ObjectMetadata and FieldMetadata. Every standard and custom object gets full CRUD operations.

**Schema generation flow:**
1. WorkspaceSchemaBuilder reads all ObjectMetadata for the workspace
2. For each object, generates: Type, Input, Filter, Sort, Connection, Edge
3. Generates Query type with `findOne{Object}` and `find{Object}s` for each object
4. Generates Mutation type with create/update/delete operations
5. Schema is cached per workspace + schema version

**Example queries:**

```graphql
# Find people with filters
query {
  findPeople(
    filter: {
      company: { name: { eq: "Acme Corp" } }
      jobTitle: { like: "%Engineer%" }
    }
    orderBy: [{ name: { firstName: AscNullsLast } }]
    first: 20
    after: "cursor_abc"
  ) {
    edges {
      node {
        id
        name { firstName lastName }
        emails
        company { name domainName { url label } }
        jobTitle
      }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
    totalCount
  }
}

# Create a person
mutation {
  createOnePerson(data: {
    name: { firstName: "Jane", lastName: "Smith" }
    emails: [{ address: "jane@acme.com", isPrimary: true }]
    companyId: "uuid-acme"
    jobTitle: "CTO"
  }) {
    id
    name { firstName lastName }
  }
}

# Update opportunity stage
mutation {
  updateOneOpportunity(
    id: "uuid-opp-1"
    data: { stage: PROPOSAL, probability: 60 }
  ) {
    id
    stage
    probability
  }
}
```

### Metadata API (`/metadata`)

Static schema for managing the data model itself. Requires DATA_MODEL permission. See @SECURITY.md#rbac.

**Key operations:**

```graphql
# Create custom object
mutation {
  createOneObjectMetadata(input: {
    nameSingular: "project"
    namePlural: "projects"
    labelSingular: "Project"
    labelPlural: "Projects"
    icon: "IconFolder"
    description: "Internal projects"
  }) {
    id
    nameSingular
  }
}

# Add custom field to object
mutation {
  createOneFieldMetadata(input: {
    objectMetadataId: "uuid-project"
    name: "budget"
    label: "Budget"
    type: CURRENCY
    description: "Project budget"
    isNullable: true
  }) {
    id
    name
    type
  }
}

# Create relation between objects
mutation {
  createOneRelationMetadata(input: {
    fromObjectMetadataId: "uuid-project"
    toObjectMetadataId: "uuid-company"
    fromFieldMetadataId: "uuid-project-company"
    toFieldMetadataId: "uuid-company-projects"
    relationType: MANY_TO_ONE
    onDeleteAction: SET_NULL
  }) {
    id
  }
}
```

### Admin Panel API (`/admin`)

Workspace administration, billing management, feature flags. Restricted to workspace owners/admins.

**Key operations:**
- Workspace settings CRUD
- Billing subscription management
- Feature flag toggling
- User/member management
- Workspace data export/import

## Dynamic REST API

Auto-generated REST endpoints mirroring the Core GraphQL API. OpenAPI specification generated from metadata.

**URL pattern:** `/api/v1/{objectNamePlural}`

```
GET    /api/v1/people              # List people (with query params for filter/sort/page)
GET    /api/v1/people/:id          # Get single person
POST   /api/v1/people              # Create person
PATCH  /api/v1/people/:id          # Update person
DELETE /api/v1/people/:id          # Soft delete person
POST   /api/v1/people/:id/restore  # Restore soft-deleted person
```

**Query parameters:**
- `filter[field][operator]=value` — e.g., `filter[jobTitle][like]=%Engineer%`
- `orderBy[field]=direction` — e.g., `orderBy[createdAt]=DescNullsLast`
- `first=20&after=cursor_abc` — Cursor pagination
- `fields=id,name,company.name` — Field selection (sparse fieldsets)

## Apollo Client Configuration

The frontend uses ApolloFactory for client creation. See @UI.md#state-management.

**Key features:**
- **Token renewal** — On 401 response, automatically refreshes access token using refresh token, then retries the request
- **Schema version** — Sends `X-Schema-Version` header. On version mismatch, refetches schema
- **Locale** — Sends `x-locale` header for i18n field labels
- **Error link** — Catches GraphQL errors, routes to error handler module. See @ERRORS.md
- **Cache** — Normalized cache with type policies for cursor-based pagination

```typescript
// Apollo Client setup pattern
const client = ApolloFactory.create({
  uri: '/api',
  tokenPair: { accessToken, refreshToken },
  onTokenRefresh: async () => { /* refresh logic */ },
  schemaVersion: currentSchemaVersion,
  locale: currentLocale,
  onUnauthenticated: () => { /* redirect to login */ },
});
```

## Webhooks

Outbound HTTP callbacks triggered by database mutations. See @RUNTIME.md for dispatch mechanics.

**Configuration:**
```graphql
mutation {
  createOneWebhook(data: {
    targetUrl: "https://example.com/webhook"
    operations: ["person.created", "person.updated", "opportunity.*.created"]
  }) {
    id
    targetUrl
    operations
  }
}
```

**Payload format:**
```json
{
  "event": "person.created",
  "workspaceId": "uuid-workspace",
  "timestamp": "2026-05-13T10:30:00Z",
  "data": {
    "id": "uuid-person",
    "name": { "firstName": "Jane", "lastName": "Smith" },
    "emails": [{ "address": "jane@acme.com", "isPrimary": true }],
    "companyId": "uuid-acme"
  }
}
```

**Retry policy:** 3 retries with exponential backoff (1s, 5s, 30s). Dead-letter after 3 failures.

## API Versioning

- GraphQL schema is versioned via `X-Schema-Version` header (content hash)
- REST API uses URL versioning (`/api/v1/`)
- Breaking change detection runs in CI via schema diff. See @TESTING.md#breaking-change-detection

## Cross-References

- @SCHEMA.md — Data model that drives API schema generation
- @ARCHITECTURE.md#twenty-orm — Query execution layer
- @SECURITY.md — Authentication and authorization
- @ERRORS.md — Error format and handling
- @RUNTIME.md#webhooks — Webhook dispatch system
