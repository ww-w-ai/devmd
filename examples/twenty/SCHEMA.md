---
devmd: schema
version: "1.0"
project: Twenty CRM
databases:
  primary:
    engine: PostgreSQL 16
    role: "Core data, metadata, workspace schemas"
    multi_tenant: true
    tenant_strategy: schema-per-workspace
  analytics:
    engine: ClickHouse
    role: Event analytics and reporting
  cache:
    engine: Redis
    role: "Cache, job queue, pub/sub"

schemas:
  public:
    description: "Core tables shared across all workspaces"
    tables:
      - user
      - workspace
      - workspace_member
      - refresh_token
      - app_token
      - billing_subscription
      - billing_entitlement
      - feature_flag
      - key_value_pair
  workspace:
    description: "Per-workspace schema (workspace_{id}). Contains all CRM data."
    naming: "workspace_{uuid}"
    provisioning: automatic on workspace creation

metadata_system:
  description: >
    Runtime-modifiable schema definitions. Users create objects and fields
    via UI or Metadata API. Changes take effect immediately.
  tables:
    - name: object_metadata
      fields: [id, workspace_id, name_singular, name_plural, label_singular, label_plural, icon, description, is_custom, is_active, is_remote, is_searchable]
    - name: field_metadata
      fields: [id, workspace_id, object_metadata_id, name, label, type, description, icon, is_custom, is_active, is_nullable, default_value, options, settings]
    - name: relation_metadata
      fields: [id, workspace_id, from_object_id, to_object_id, from_field_id, to_field_id, relation_type, on_delete_action]
    - name: index_metadata
      fields: [id, workspace_id, object_metadata_id, name, columns, index_type, is_unique]
    - name: view_metadata
      fields: [id, workspace_id, object_metadata_id, name, type, icon, position, is_compact, filter_groups, sorts, fields, kanban_column_id, group_by_id]

field_types:
  primitives:
    - TEXT
    - NUMBER
    - BOOLEAN
    - DATE_TIME
    - DATE
    - UUID
    - ENUM
    - MULTI_ENUM
    - RATING
    - POSITION
    - RAW_JSON
    - RICH_TEXT
    - ARRAY
  composites:
    - name: FULL_NAME
      columns: [nameFirstName, nameLastName]
    - name: ADDRESS
      columns: [addressStreet1, addressStreet2, addressCity, addressState, addressPostcode, addressCountry, addressLat, addressLng]
    - name: CURRENCY
      columns: [amountMicros, currencyCode]
    - name: LINK
      columns: [linkUrl, linkLabel]
    - name: LINKS
      columns: [linksJson]
      description: "Array of {url, label} stored as JSONB"
    - name: EMAILS
      columns: [emailsJson]
      description: "Array of {address, isPrimary} stored as JSONB"
    - name: PHONES
      columns: [phonesJson]
      description: "Array of {number, countryCode, isPrimary} stored as JSONB"
    - name: ACTOR
      columns: [actorSource, actorWorkspaceMemberId, actorName]
      description: "Who performed an action (manual, api, import, workflow)"

relation_types:
  - ONE_TO_MANY
  - MANY_TO_ONE
  - MANY_TO_MANY

on_delete_actions:
  - SET_NULL
  - CASCADE
  - RESTRICT

migration_rules:
  - rule: "Standard objects cannot be deleted"
  - rule: "Standard fields on standard objects cannot be deleted"
  - rule: "Custom fields on standard objects can be added/deleted"
  - rule: "Custom objects can be created/deleted at runtime"
  - rule: "Field type changes require data migration"
  - rule: "Relation changes cascade to both sides"
  - rule: "Index metadata changes apply to workspace schemas immediately"
---

# Twenty CRM Schema

## Multi-Tenant Architecture

Each workspace gets a dedicated PostgreSQL schema. See @GLOSSARY.md#workspace-schema.

```
PostgreSQL
├── public schema (shared)
│   ├── user                    # Global user accounts
│   ├── workspace               # Workspace definitions
│   ├── workspace_member        # User ↔ Workspace mapping
│   ├── object_metadata         # Object definitions (all workspaces)
│   ├── field_metadata          # Field definitions
│   ├── relation_metadata       # Relation definitions
│   ├── index_metadata          # Index definitions
│   ├── view                    # View configurations
│   ├── refresh_token           # Auth tokens
│   ├── app_token               # API keys
│   ├── billing_subscription    # Billing state
│   ├── billing_entitlement     # Feature entitlements
│   ├── feature_flag            # Feature toggles
│   └── key_value_pair          # Generic KV store
│
├── workspace_abc123 schema
│   ├── person                  # Contacts
│   ├── company                 # Accounts
│   ├── opportunity             # Deals
│   ├── note                    # Notes
│   ├── note_target             # Note ↔ Record association
│   ├── task                    # Tasks
│   ├── task_target             # Task ↔ Record association
│   ├── attachment              # File attachments
│   ├── dashboard               # Dashboards
│   ├── workflow                # Automation workflows
│   ├── workflow_trigger        # Workflow triggers
│   ├── workflow_action         # Workflow actions
│   ├── connected_account       # Email/calendar accounts
│   ├── message                 # Synced emails
│   ├── message_channel         # Email sync channels
│   ├── message_participant     # Email participants
│   ├── message_thread          # Email threads
│   ├── calendar_event          # Synced calendar events
│   ├── calendar_event_participant
│   ├── calendar_channel        # Calendar sync channels
│   ├── blocklist               # Email blocklist
│   ├── favorite                # Bookmarked records
│   ├── webhook                 # Webhook configurations
│   ├── audit_log               # Change audit trail
│   ├── _custom_object_*        # User-created objects
│   └── _relation_*             # Many-to-many join tables
│
├── workspace_def456 schema     # Another tenant
│   └── ... (same structure)
└── ...
```

## Standard Objects

### Person (Contact)

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| name | FULL_NAME | First + last name (composite) |
| emails | EMAILS | Array of email addresses |
| phones | PHONES | Array of phone numbers |
| city | TEXT | City |
| jobTitle | TEXT | Job title |
| linkedinLink | LINK | LinkedIn profile URL |
| xLink | LINK | X/Twitter profile URL |
| avatarUrl | TEXT | Profile picture URL |
| position | POSITION | Sort position |
| companyId | UUID | FK → Company |
| createdBy | ACTOR | Who created this record |
| createdAt | DATE_TIME | Creation timestamp |
| updatedAt | DATE_TIME | Last update timestamp |
| deletedAt | DATE_TIME | Soft delete timestamp |

**Relations:**
- `company` → Company (MANY_TO_ONE)
- `noteTargets` → NoteTarget (ONE_TO_MANY)
- `taskTargets` → TaskTarget (ONE_TO_MANY)
- `favorites` → Favorite (ONE_TO_MANY)
- `attachments` → Attachment (ONE_TO_MANY)
- `messageParticipants` → MessageParticipant (ONE_TO_MANY)
- `calendarEventParticipants` → CalendarEventParticipant (ONE_TO_MANY)

### Company (Account)

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| name | TEXT | Company name |
| domainName | LINK | Primary website |
| address | ADDRESS | Headquarters address (composite) |
| employees | NUMBER | Employee count |
| annualRevenue | CURRENCY | Annual revenue (composite) |
| idealCustomerProfile | BOOLEAN | ICP flag |
| linkedinLink | LINK | LinkedIn page |
| xLink | LINK | X/Twitter page |
| position | POSITION | Sort position |
| createdBy | ACTOR | Who created this record |
| createdAt | DATE_TIME | Creation timestamp |
| updatedAt | DATE_TIME | Last update timestamp |
| deletedAt | DATE_TIME | Soft delete timestamp |

**Relations:**
- `people` → Person (ONE_TO_MANY)
- `opportunities` → Opportunity (ONE_TO_MANY)
- `noteTargets` → NoteTarget (ONE_TO_MANY)
- `taskTargets` → TaskTarget (ONE_TO_MANY)
- `favorites` → Favorite (ONE_TO_MANY)
- `attachments` → Attachment (ONE_TO_MANY)

### Opportunity (Deal)

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| name | TEXT | Deal name |
| amount | CURRENCY | Deal value |
| closeDate | DATE | Expected close date |
| probability | NUMBER | Win probability (0-100) |
| stage | ENUM | Pipeline stage |
| pointOfContactId | UUID | FK → Person (primary contact) |
| companyId | UUID | FK → Company |
| position | POSITION | Sort position in kanban |
| createdBy | ACTOR | Who created this record |
| createdAt | DATE_TIME | Creation timestamp |
| updatedAt | DATE_TIME | Last update timestamp |
| deletedAt | DATE_TIME | Soft delete timestamp |

**Default pipeline stages:**
1. `QUALIFICATION` — New lead being evaluated
2. `MEETING` — Discovery/demo meeting scheduled
3. `PROPOSAL` — Proposal sent
4. `NEGOTIATION` — Terms being negotiated
5. `CLOSED_WON` — Deal won
6. `CLOSED_LOST` — Deal lost

### Connected Account

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| handle | TEXT | Email address |
| provider | ENUM | GOOGLE, MICROSOFT, GENERIC_IMAP |
| accessToken | TEXT | OAuth access token (encrypted) |
| refreshToken | TEXT | OAuth refresh token (encrypted) |
| accountOwnerId | UUID | FK → Workspace Member |
| lastSyncHistoryId | UUID | Last sync cursor |
| authFailedAt | DATE_TIME | Last auth failure |

### Workflow

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| name | TEXT | Workflow name |
| isActive | BOOLEAN | Enabled flag |
| trigger | RAW_JSON | Trigger configuration |
| actions | RAW_JSON | Action sequence configuration |
| statuses | RAW_JSON | Execution status tracking |
| position | POSITION | Sort position |

## Composite Fields

Composite fields map a single logical field to multiple database columns. See @GLOSSARY.md#composite-field.

```
FULL_NAME "John Doe"
  → nameFirstName: "John"
  → nameLastName: "Doe"

ADDRESS "123 Main St, San Francisco, CA 94105, US"
  → addressStreet1: "123 Main St"
  → addressStreet2: null
  → addressCity: "San Francisco"
  → addressState: "CA"
  → addressPostcode: "94105"
  → addressCountry: "US"
  → addressLat: 37.7749
  → addressLng: -122.4194

CURRENCY "$50,000.00 USD"
  → amountMicros: 50000000000
  → currencyCode: "USD"

ACTOR "Created by John via API"
  → actorSource: "API"
  → actorWorkspaceMemberId: "uuid-123"
  → actorName: "John Doe"
```

The twenty-orm query builders handle composite field expansion automatically. When filtering by `name`, the query builder generates `WHERE nameFirstName LIKE ... OR nameLastName LIKE ...`. See @ARCHITECTURE.md#twenty-orm.

## Custom Objects

Users create custom objects at runtime via the Metadata API or UI. See @GLOSSARY.md#custom-object.

**Creation flow:**
1. User defines object via Metadata API (`createOneObjectMetadata` mutation)
2. ObjectMetadata record created in public schema
3. Table created in workspace schema (`_custom_{name}`)
4. Default fields added (id, createdAt, updatedAt, deletedAt, position, createdBy)
5. WorkspaceSchemaBuilder regenerates GraphQL schema
6. Object appears in navigation, supports views, filters, GraphQL queries

**Custom field creation:**
1. User defines field via Metadata API (`createOneFieldMetadata` mutation)
2. FieldMetadata record created in public schema
3. Column(s) added to workspace table (composite fields add multiple columns)
4. GraphQL schema regenerated
5. Field appears in views, filters, sorts

## Indexes

Default indexes on all objects:
- Primary key (id)
- `deletedAt` (soft delete queries)
- `createdAt` (sort by creation date)
- `updatedAt` (change detection)

Custom indexes defined via IndexMetadata. Types: BTREE (default), GIN (for JSONB/array), GiST (for full-text/spatial).

## ClickHouse Analytics

Event data flows to ClickHouse for analytics queries:
- Page views, feature usage
- API call metrics
- Workflow execution metrics
- User activity aggregates

ClickHouse is append-only. No updates or deletes. Queries use materialized views for common aggregations.

## Cross-References

- @ARCHITECTURE.md#twenty-orm — How the ORM reads and applies schema
- @API.md#core-api — How schema maps to GraphQL types
- @SECURITY.md#rbac — Permission enforcement at schema level
- @GLOSSARY.md — Term definitions for all objects and field types
