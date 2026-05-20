---
devmd: flows
version: "1.0"
project: Twenty CRM

flows:
  - name: create-custom-object
    trigger: user
    touchpoints: [ui-settings, metadata-api, schema-builder, graphql, navigation]
    complexity: medium
  - name: email-sync
    trigger: cron
    touchpoints: [connected-account, imap, worker, message-parsing, participant-matching, timeline]
    complexity: high
  - name: record-crud-with-permissions
    trigger: user
    touchpoints: [ui, apollo-client, graphql-resolver, permission-guard, twenty-orm, rls, sse]
    complexity: high
  - name: workflow-execution
    trigger: event
    touchpoints: [trigger-event, condition-evaluator, action-executor, worker, audit-log]
    complexity: medium
  - name: workspace-provisioning
    trigger: signup
    touchpoints: [auth, postgres-schema, metadata-seeding, default-views]
    complexity: high
---

# Twenty CRM — End-to-End Flows

Each flow documents both the UX perspective (what the user sees) and the Data perspective (what the system does). Cross-references point to the DevMD file that owns the detail.

## 1. Create Custom Object Flow

A user creates a new data type at runtime without writing code. See @SCHEMA.md#custom-objects, @API.md#metadata-api.

### UX Perspective

```
1. User navigates to Settings → Data Model
2. Clicks "New Object"
3. Fills form: name (singular/plural), icon, description
4. Object card appears in data model list
5. User clicks "Add Field" → selects type (Text, Number, Currency, etc.)
6. Repeats for each field
7. User clicks "Add Relation" → selects target object + relation type
8. Returns to main nav → new object appears in sidebar
9. Clicks new object → empty list view with configured fields as columns
10. Can immediately create records, apply filters, save views
```

### Data Perspective

```
Settings UI (twenty-front)
  │  User fills object form
  ▼
Apollo Client → Metadata API (/metadata)
  │  mutation: createOneObjectMetadata
  │  Payload: { nameSingular, namePlural, labelSingular, labelPlural, icon, description }
  ▼
MetadataResolver (twenty-server)
  │  Validates naming (no collision with standard objects)
  │  Validates permissions (DATA_MODEL required). See @SECURITY.md#rbac
  ▼
ObjectMetadataService
  │  1. Creates ObjectMetadata record in public schema
  │  2. Creates table _custom_{name} in workspace schema
  │  3. Adds default fields: id, createdAt, updatedAt, deletedAt, position, createdBy
  │  4. Creates default indexes (PK, deletedAt, createdAt)
  ▼
WorkspaceSchemaBuilder
  │  Regenerates GraphQL schema for this workspace
  │  New type, input, filter, sort, connection, edge types added
  │  Schema version incremented
  ▼
SSE Event → Apollo Cache
  │  schema.changed event sent to connected clients
  │  Frontend refetches schema, updates navigation
  ▼
Object visible in sidebar, ready for CRUD
```

### Field Addition Sub-Flow

```
User clicks "Add Field" on object detail
  │
  ▼
Metadata API: createOneFieldMetadata
  │  Payload: { objectMetadataId, name, label, type, icon, isNullable, defaultValue }
  ▼
FieldMetadataService
  │  1. Creates FieldMetadata record in public schema
  │  2. Adds column(s) to workspace table
  │     - Simple types: 1 column (e.g., TEXT → VARCHAR)
  │     - Composite types: N columns (e.g., CURRENCY → amountMicros + currencyCode)
  │     See @SCHEMA.md#composite-fields
  │  3. Applies default value if specified
  ▼
WorkspaceSchemaBuilder regenerates GraphQL schema
  ▼
Field appears in views, filters, sorts, GraphQL queries
```

### Relation Addition Sub-Flow

```
User clicks "Add Relation"
  │  Selects target object, relation type (One-to-Many, Many-to-Many)
  ▼
Metadata API: createOneRelationMetadata
  │  Payload: { fromObjectMetadataId, toObjectMetadataId, relationType, onDeleteAction }
  ▼
RelationMetadataService
  │  MANY_TO_ONE: adds FK column to source table
  │  MANY_TO_MANY: creates join table _relation_{from}_{to}
  │  Creates FieldMetadata on both sides (forward + inverse)
  ▼
GraphQL schema updated with relation fields
  ▼
UI shows relation picker on record detail. See @UI.md#page-types
```

---

## 2. Email Sync Flow

Connect an email account and automatically sync messages to CRM timelines. See @RUNTIME.md#email-calendar-sync, @SCHEMA.md#connected-account.

### UX Perspective

```
1. User navigates to Settings → Accounts → Connected Accounts
2. Clicks "Connect Account" → selects provider (Gmail, Outlook, IMAP)
3. OAuth redirect to provider (Gmail/Outlook) or IMAP credentials form
4. On success: account appears as "Connected" with sync status
5. User navigates to a Person record → Activity Timeline
6. Emails exchanged with that person appear automatically
7. Clicking an email opens the message with thread context
8. User can reply inline (drafts via connected account)
```

### Data Perspective

```
User initiates connection
  │
  ├── Gmail/Outlook: OAuth 2.0 flow
  │   │  Redirect to provider authorization page
  │   │  User grants email read/send scopes
  │   │  Callback with authorization code
  │   │  Exchange for access_token + refresh_token
  │   └── Tokens encrypted and stored in ConnectedAccount record
  │
  ├── Generic IMAP: Credential entry
  │   │  User provides host, port, username, password
  │   │  Server validates IMAP connection
  │   └── Credentials encrypted and stored in ConnectedAccount record
  │
  ▼
ConnectedAccount created in workspace schema
  │  provider: GOOGLE | MICROSOFT | GENERIC_IMAP
  │  handle: user@example.com
  │  accessToken: encrypted
  │  refreshToken: encrypted
  │  lastSyncHistoryId: null (first sync)
  ▼
Cron (every 5 min) → enqueues email-sync job. See @RUNTIME.md#cron-job-schedule
  ▼
Worker: email-sync queue
  │
  ├── Connect to IMAP server using stored credentials
  │   │  Gmail: imap.gmail.com:993 with OAuth2 XOAUTH2
  │   │  Outlook: outlook.office365.com:993 with OAuth2
  │   │  Generic: user-specified host:port with password
  │
  ├── Fetch new messages since lastSyncHistoryId
  │   │  IMAP SEARCH since last UID
  │   │  Batch fetch: headers + body + attachments
  │
  ├── For each message:
  │   │
  │   ├── Parse email (headers, body HTML→text, attachments)
  │   │
  │   ├── Thread detection:
  │   │   │  Match by In-Reply-To / References headers
  │   │   │  Fallback: subject line matching
  │   │   └── Create or link to MessageThread record
  │   │
  │   ├── Participant matching:
  │   │   │  For each From/To/Cc/Bcc address:
  │   │   │    Search Person records by email. See @SCHEMA.md#person
  │   │   │    If match → create MessageParticipant with personId
  │   │   │    If no match → create MessageParticipant with email only
  │   │   └── Check Blocklist — skip blocked addresses. See @GLOSSARY.md#blocklist
  │   │
  │   ├── Create Message record:
  │   │   │  subject, body, receivedAt, messageExternalId
  │   │   │  Link to MessageThread, MessageChannel
  │   │   └── Store attachments (local filesystem or S3). See @INFRA.md
  │   │
  │   └── Emit message.created event → SSE → UI timeline updates
  │
  ├── Update lastSyncHistoryId on ConnectedAccount
  │
  └── Error handling:
      ├── Auth failure (401/expired token):
      │   │  Attempt token refresh (OAuth providers)
      │   │  If refresh fails: set authFailedAt, notify user via UI badge
      │   └── Skip account until user re-authenticates
      ├── Rate limit (429):
      │   └── Retry with exponential backoff. See @RUNTIME.md#queue-configuration
      └── Network/IMAP error:
          └── Log error, retry on next cron cycle. See @LOGGING.md
```

---

## 3. Record CRUD with Permissions

Full lifecycle of creating, reading, updating, and deleting a record with 4-layer RBAC enforcement. See @SECURITY.md#rbac, @API.md#core-api.

### UX Perspective

```
CREATE:
1. User clicks "+" button on People list view
2. Inline row appears with editable fields
3. User types name, email, selects company from dropdown
4. Press Enter or click away → record saved
5. New row appears in table with all fields populated

READ:
1. User sees list of records filtered by current View
2. Clicks a record → right panel opens with full detail
3. Activity timeline shows notes, tasks, emails, events
4. Related records (company, opportunities) shown in sidebar

UPDATE:
1. User clicks a field value on record detail
2. Field becomes editable (inline edit)
3. User modifies value → auto-saves on blur
4. Optimistic update: UI reflects change immediately
5. If server rejects (permission denied): UI reverts, shows error toast

DELETE:
1. User selects record(s) → clicks "Delete" in bulk action bar
2. Confirmation dialog appears
3. Records soft-deleted (deletedAt set). See @GLOSSARY.md#record-soft-delete
4. Records move to Trash view (30-day retention)
5. User can restore from Trash within retention period
```

### Data Perspective

```
User action in UI (e.g., create person)
  │
  ▼
Apollo Client (twenty-front)
  │  Builds GraphQL mutation
  │  Adds optimistic response to cache
  │  Sends request with Authorization header (JWT)
  │  Includes traceparent header. See @LOGGING.md#trace-propagation
  ▼
GraphQL Yoga (twenty-server, /api endpoint)
  │
  ├── Step 1: Authentication
  │   │  WorkspaceAuthGuard extracts JWT from Authorization header
  │   │  Validates signature, expiry, workspace membership
  │   │  Attaches { userId, workspaceId, role, permissions } to request context
  │   └── On failure: return UNAUTHENTICATED error. See @ERRORS.md
  │
  ├── Step 2: Object Permission Check
  │   │  Resolve which object type is being accessed (e.g., Person)
  │   │  Check role's object-level permission: CREATE_PERSON / READ_PERSON / etc.
  │   └── On failure: return OBJECT_PERMISSION_DENIED error
  │
  ├── Step 3: Field Permission Check
  │   │  For each field in the mutation input or query selection:
  │   │    Check role's field-level permission: READ_{FIELD} / UPDATE_{FIELD}
  │   │  Strip unauthorized fields from input (write) or response (read)
  │   └── On all fields denied: return FIELD_PERMISSION_DENIED error
  │
  ├── Step 4: twenty-orm Query Execution
  │   │  WorkspaceEntityManager builds SQL query
  │   │  Sets search_path to workspace schema: SET search_path TO workspace_{id}
  │   │
  │   │  Row-Level Security (Layer 4):
  │   │    RLS predicates injected as WHERE clauses based on user role
  │   │    Example: Guest role → WHERE createdBy.actorWorkspaceMemberId = :userId
  │   │
  │   │  Composite field expansion:
  │   │    Input { name: { firstName: "Jane" } }
  │   │    → SQL: INSERT ... (nameFirstName) VALUES ('Jane')
  │   │    See @SCHEMA.md#composite-fields
  │   │
  │   └── Execute against PostgreSQL
  │
  ├── Step 5: Event Emission
  │   │  twenty-orm emits mutation event: person.created / person.updated / person.deleted
  │   │
  │   ├── SSE channel: publish to Redis pub/sub → stream to connected clients
  │   │   Client Apollo cache receives event → updates/invalidates affected queries
  │   │
  │   ├── Webhook dispatcher: match against registered webhooks → enqueue dispatch jobs
  │   │   See @RUNTIME.md#webhook-dispatch, @API.md#webhooks
  │   │
  │   ├── Workflow engine: evaluate active workflows for this trigger type
  │   │   See @RUNTIME.md#workflow-engine
  │   │
  │   └── Audit log: write AuditLog record with operation, fields changed, actor
  │
  └── Step 6: Response
      │  Return GraphQL response with created/updated record
      │  Include trace ID in response headers
      ▼
Apollo Client receives response
  │  Merges with optimistic cache (or reverts on error)
  │  UI reflects final state
```

---

## 4. Workflow Execution Flow

A user-defined automation triggers and executes a sequence of actions. See @RUNTIME.md#workflow-engine, @GLOSSARY.md#workflow.

### UX Perspective

```
SETUP:
1. User navigates to Settings → Workflows
2. Clicks "New Workflow"
3. Selects trigger: "When an Opportunity is updated"
4. Adds condition: "Stage changed to CLOSED_WON"
5. Adds action 1: "Create Task — Send congratulations email"
6. Adds action 2: "Call Webhook — notify Slack channel"
7. Saves and activates workflow

EXECUTION (invisible to user):
8. Sales rep moves an Opportunity to CLOSED_WON stage
9. Task appears on the Opportunity timeline within seconds
10. Slack channel receives notification

MONITORING:
11. User views Workflow → Execution History
12. Each execution shows: trigger record, actions taken, status (success/failed), duration
```

### Data Perspective

```
Database mutation event: opportunity.updated
  │  Changed fields include: stage → CLOSED_WON
  │  Event emitted by twenty-orm. See @RUNTIME.md#event-emission
  ▼
WorkflowEngine (event listener in twenty-server)
  │
  ├── Load all active Workflows with trigger type = record.updated
  │   AND trigger object = opportunity
  │
  ├── For each matching workflow:
  │   │
  │   ├── Evaluate conditions:
  │   │   │  Parse condition JSON: { field: "stage", operator: "eq", value: "CLOSED_WON" }
  │   │   │  Compare against actual changed values
  │   │   └── If conditions not met → skip this workflow
  │   │
  │   └── Conditions met → enqueue workflow-execution job
  │       Payload: { workflowId, triggerId, recordId, changedFields, workspaceId }
  │
  ▼
Worker: workflow-execution queue. See @RUNTIME.md#queue-configuration
  │
  ├── Load workflow actions array from Workflow record
  │
  ├── Execute actions sequentially:
  │   │
  │   ├── Action 1: create-task
  │   │   │  Resolve template variables: {{record.name}} → "Acme Enterprise Deal"
  │   │   │  Create Task record via twenty-orm
  │   │   │  Link to Opportunity via TaskTarget
  │   │   └── Log: action_completed { type: create-task, taskId: uuid }
  │   │
  │   ├── Action 2: call-webhook
  │   │   │  Resolve payload template with record data
  │   │   │  POST to configured URL (Slack webhook)
  │   │   │  Sign payload with HMAC. See @RUNTIME.md#webhook-signature
  │   │   ├── On 2xx: log success
  │   │   ├── On 5xx: retry with backoff (max 3 attempts)
  │   │   └── On max retries: mark action as failed, continue to next action
  │   │
  │   └── (Additional actions if configured: wait, condition branch, etc.)
  │
  ├── Update Workflow execution status:
  │   │  All actions succeeded → status: completed
  │   │  Some actions failed → status: partial
  │   │  Critical action failed → status: failed
  │
  └── Write audit log entry. See @LOGGING.md
```

---

## 5. Workspace Provisioning Flow

A new user signs up and gets a fully functional CRM workspace. See @SCHEMA.md#multi-tenant, @SECURITY.md#authentication.

### UX Perspective

```
1. User visits twenty.com/signup
2. Enters email and password (or clicks Google/Microsoft SSO)
3. Email verification (if password signup)
4. Workspace creation form: workspace name, logo (optional)
5. Loading screen: "Setting up your workspace..." (2-5 seconds)
6. Onboarding wizard:
   a. Invite team members (optional, skip for now)
   b. Connect email account (optional, skip for now)
   c. Import data from CSV or other CRM (optional, skip for now)
7. Lands on People list view — workspace is ready
```

### Data Perspective

```
Signup request (POST /auth/signup)
  │
  ├── Validate input:
  │   │  Email format, password strength (bcrypt hash). See @SECURITY.md#password-policy
  │   │  Check email uniqueness in User table (public schema)
  │   └── On validation failure: return error. See @ERRORS.md
  │
  ├── Create User record (public schema):
  │   │  id, email, passwordHash, isEmailVerified: false
  │   └── Send email verification token (24h expiry). See @SECURITY.md#tokens
  │
  ├── (Email verification step — user clicks link)
  │   │  Validate token → set isEmailVerified: true
  │
  ├── Create Workspace record (public schema):
  │   │  id, displayName, logo, subdomain
  │   │  status: CREATING
  │
  ├── Create WorkspaceMember record:
  │   │  userId, workspaceId, role: OWNER
  │
  ▼
Workspace provisioning (async or synchronous depending on deployment)
  │
  ├── Step 1: Create PostgreSQL schema
  │   │  CREATE SCHEMA workspace_{id}
  │   │  SET search_path TO workspace_{id}
  │   └── On failure: set workspace status to FAILED, alert. See @LOGGING.md
  │
  ├── Step 2: Run workspace migrations
  │   │  Create all standard object tables:
  │   │    person, company, opportunity, note, note_target, task, task_target,
  │   │    attachment, dashboard, workflow, workflow_trigger, workflow_action,
  │   │    connected_account, message, message_channel, message_participant,
  │   │    message_thread, calendar_event, calendar_event_participant,
  │   │    calendar_channel, blocklist, favorite, webhook, audit_log
  │   │  Create indexes on all tables. See @SCHEMA.md#indexes
  │   └── On failure: DROP SCHEMA workspace_{id} CASCADE, set status FAILED
  │
  ├── Step 3: Seed metadata
  │   │  Create ObjectMetadata records for all standard objects
  │   │  Create FieldMetadata records for all standard fields
  │   │  Create RelationMetadata records for all standard relations
  │   │  Create IndexMetadata records
  │   └── These records drive GraphQL schema generation. See @API.md#core-api
  │
  ├── Step 4: Create default views
  │   │  People: default table view with name, email, company, jobTitle columns
  │   │  Companies: default table view with name, domain, employees columns
  │   │  Opportunities: default kanban view grouped by stage
  │   └── See @UI.md#views, @GLOSSARY.md#view
  │
  ├── Step 5: Generate GraphQL schema
  │   │  WorkspaceSchemaBuilder reads all metadata
  │   │  Generates types, queries, mutations for this workspace
  │   └── Caches schema by workspace + version
  │
  ├── Step 6: Set workspace status to READY
  │
  ▼
Issue tokens
  │  Login Token → Access Token + Refresh Token
  │  Workspace context embedded in JWT payload. See @SECURITY.md#tokens
  ▼
Redirect to /objects/people (default landing page)
  │  Frontend loads with full schema, navigation, empty views
  │  Onboarding wizard overlays on first visit
```

### Failure Recovery

```
Provisioning failure at any step:
  │
  ├── Step 1-2 failure (schema/migration):
  │   │  DROP SCHEMA IF EXISTS workspace_{id} CASCADE
  │   │  Set workspace status: FAILED
  │   │  User sees: "Workspace creation failed. Please try again."
  │   └── Retry creates a fresh schema from scratch
  │
  ├── Step 3-5 failure (seeding/schema generation):
  │   │  Workspace schema exists but is incomplete
  │   │  Set workspace status: FAILED
  │   │  Admin can trigger re-seeding via CLI:
  │   │    nx run twenty-server:command -- workspace:seed --workspaceId={id}
  │   └── See @CLAUDE.md#development-workflow
  │
  └── Monitoring:
      │  workspace_provisioning_duration_seconds histogram. See @LOGGING.md#key-metrics
      │  workspace_provisioning_failure_total counter
      └── Alert on failure rate > 1%. See @ERRORS.md#metrics
```

---

## Flow Dependencies

```
Workspace Provisioning
  └── enables all other flows (must complete first)

Create Custom Object
  └── requires: DATA_MODEL permission. See @SECURITY.md#rbac
  └── triggers: schema regeneration → affects Record CRUD queries

Email Sync
  └── requires: Connected Account (OAuth or IMAP credentials)
  └── triggers: message.created events → may trigger Workflow Execution

Record CRUD
  └── requires: object + field + row permissions
  └── triggers: events → Workflow Execution, Webhook Dispatch, SSE

Workflow Execution
  └── requires: active workflow + matching trigger event
  └── may trigger: Record CRUD (create-record, update-record actions)
  └── may trigger: Webhook Dispatch (call-webhook action)
```

## Cross-References

- @SCHEMA.md — Object definitions, composite fields, multi-tenant architecture
- @API.md — GraphQL schema generation, mutations, webhooks
- @SECURITY.md — Authentication, RBAC 4-layer enforcement
- @RUNTIME.md — Worker queues, email sync, workflow engine, SSE
- @ERRORS.md — Error handling at each step
- @LOGGING.md — Trace propagation through flows
- @UI.md — Page types and views referenced in UX perspective
- @GLOSSARY.md — Term definitions for all domain concepts
