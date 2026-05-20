---
devmd: glossary
version: "1.0"
project: Twenty CRM
domain: crm

terms:
  # --- CRM Domain ---
  - term: Person
    aka: [Contact, Lead]
    definition: >
      An individual tracked in the CRM. Has name (FullName composite), email,
      phone, company association, city, job title. Standard object.
    see: "@SCHEMA.md#standard-objects"

  - term: Company
    aka: [Account, Organization]
    definition: >
      A business entity. Has name, domain URL, address (Address composite),
      employee count, annual revenue, ICP flag. Standard object.
    see: "@SCHEMA.md#standard-objects"

  - term: Opportunity
    aka: [Deal]
    definition: >
      A potential sale tracked through pipeline stages (e.g., Qualification →
      Meeting → Proposal → Closed Won/Lost). Has amount, close date, probability,
      point of contact, associated company. Standard object.
    see: "@SCHEMA.md#standard-objects"

  - term: Pipeline
    definition: >
      A sequence of stages an Opportunity moves through. Each stage has a
      position, color, and probability percentage. Default pipeline has 5 stages.

  - term: Note
    definition: >
      Rich text content (TipTap/BlockNote editor) attached to a Person, Company,
      or Opportunity via NoteTarget relation. Supports markdown, mentions, embeds.

  - term: Task
    definition: >
      An action item with assignee, due date, status (Open/Closed), and body.
      Attached to records via TaskTarget relation. Used for follow-ups, not
      project management.

  - term: Attachment
    definition: >
      A file uploaded to a record. Stored on local filesystem or S3.
      Has name, type, path, author.

  - term: View
    definition: >
      A saved configuration of filters, sorts, groups, and visible fields for
      an object. Views can be personal or shared. Supports table, kanban,
      and calendar layouts.
    see: "@UI.md#views"

  - term: Connected Account
    definition: >
      An external email (IMAP) or calendar (CalDAV) account linked to a
      workspace member. Enables message and event sync. Handles: Gmail,
      Outlook, generic IMAP.
    see: "@RUNTIME.md#email-calendar-sync"

  - term: Message
    definition: >
      An email message synced from a connected account. Linked to Person/Company
      via participant matching. Stored with thread structure.

  - term: Calendar Event
    definition: >
      A calendar entry synced from a connected account. Linked to participants
      (Person records) automatically.

  - term: Dashboard
    definition: >
      A configurable display of charts, metrics, and widgets. Uses Nivo charts
      library. Widgets can query any object.

  - term: Workflow
    definition: >
      An automation rule: trigger → conditions → actions. Triggers include
      record creation, update, deletion, or scheduled time. Actions include
      send email, create record, update field, call webhook.
    see: "@RUNTIME.md#workflow-engine"

  # --- Technical Domain ---
  - term: Workspace
    definition: >
      A tenant in the multi-tenant system. Each workspace gets its own
      PostgreSQL schema (e.g., workspace_abc123). All data is isolated.
      A user can belong to multiple workspaces.
    see: "@SCHEMA.md#multi-tenant"

  - term: Workspace Member
    definition: >
      A user's membership in a specific workspace. Has role (Owner, Admin,
      Member, Guest), permissions, avatar, name within that workspace.

  - term: Custom Object
    definition: >
      A user-defined data type created at runtime via the metadata API.
      Stored as ObjectMetadata. Gets its own table in the workspace schema,
      appears in navigation, supports views, filters, GraphQL queries.
    see: "@SCHEMA.md#custom-objects"

  - term: Field Metadata
    definition: >
      Definition of a field on an object (standard or custom). Includes name,
      type, label, icon, description, default value, validation options.
      Composite types (FullName, Address, Currency, Link) flatten to multiple
      DB columns.
    see: "@SCHEMA.md#field-types"

  - term: Object Metadata
    definition: >
      Definition of an object (table) in the metadata system. Includes name,
      label, icon, description, fields, relations, indexes. Drives GraphQL
      schema generation and UI rendering.

  - term: Relation Metadata
    definition: >
      Definition of a relationship between two objects. Types: ONE_TO_MANY,
      MANY_TO_ONE, MANY_TO_MANY. Creates foreign key columns and join tables
      automatically.

  - term: Index Metadata
    definition: >
      Definition of a database index on an object. Includes columns, type
      (BTREE, GIN, GiST), uniqueness constraint. Applied to workspace schemas.

  - term: View Metadata
    definition: >
      Saved view configuration stored as metadata. Includes filter groups,
      sort definitions, field visibility/width, kanban column field, group-by field.

  - term: Composite Field
    definition: >
      A logical field that maps to multiple database columns. Examples:
      FullName → nameFirstName + nameLastName; Address → addressStreet1 +
      addressStreet2 + addressCity + addressState + addressPostcode + addressCountry;
      Currency → amountMicros + currencyCode.
    see: "@SCHEMA.md#composite-fields"

  - term: Standard Object
    definition: >
      A built-in object shipped with Twenty. Cannot be deleted but can have
      custom fields added. Examples: Person, Company, Opportunity, Note, Task.
    see: "@SCHEMA.md#standard-objects"

  - term: Engine Layer
    definition: >
      The core infrastructure layer in twenty-server that provides cross-cutting
      concerns: caching, dataloaders, guards, interceptors, workspace management,
      and the custom ORM (twenty-orm).
    see: "@ARCHITECTURE.md#engine-layer"

  - term: twenty-orm
    definition: >
      Custom ORM extending TypeORM. Provides WorkspaceEntityManager (permission-aware),
      WorkspaceRepository, custom query builders, composite field handling,
      and event emission on all mutations. Enforces RBAC at query time.
    see: "@ARCHITECTURE.md#twenty-orm"

  - term: Workspace Schema
    definition: >
      A PostgreSQL schema dedicated to a single workspace tenant. Named
      workspace_{id}. Contains all object tables, indexes, and data for that
      tenant. Core/metadata tables live in the public schema.
    see: "@SCHEMA.md#multi-tenant"

  - term: Blocklist
    definition: >
      A list of email addresses or domains blocked from message sync.
      Prevents spam or unwanted contacts from being imported.

  - term: Favorite
    definition: >
      A user's bookmarked record (Person, Company, Opportunity, or custom object).
      Appears in the left sidebar for quick access.

  - term: Activity
    definition: >
      A timeline entry on a record. Can be a Note, Task, Email, Calendar Event,
      or system event (field change, stage change). Provides full audit trail.

  - term: Webhook
    definition: >
      An HTTP callback triggered by database changes. Configured per workspace.
      Sends JSON payload with operation type, object name, and record data.
    see: "@API.md#webhooks"

  - term: Skill
    aka: [AI Skill]
    definition: >
      A defined AI capability using defineSkill() from twenty-sdk. Skills are
      tools that AI agents can invoke. Examples: search contacts, summarize
      emails, enrich company data.
    see: "@AGENTS.md#skills"

  - term: Agent
    aka: [AI Agent]
    definition: >
      An autonomous AI entity using defineAgent() from twenty-sdk. Agents
      have instructions, tools (skills), and can execute multi-step workflows.
    see: "@AGENTS.md#agents"
---

# Twenty CRM Glossary

This glossary defines the ubiquitous language for the Twenty CRM domain. All code, documentation, API naming, and UI labels must use these terms consistently.

## Usage Rules

1. **Person**, not "Contact" or "Lead" — in code, API, and UI
2. **Company**, not "Account" or "Organization" — in code, API, and UI
3. **Opportunity**, not "Deal" — in code, API, and UI (UI may show "Deal" as display label)
4. **Workspace**, not "Tenant" or "Organization" — always Workspace
5. **Object**, not "Entity" or "Table" — when referring to the metadata-driven data model
6. **Field**, not "Column" or "Property" — when referring to metadata fields
7. **View**, not "List" or "Filter preset" — always View

## Domain Map

```
Workspace (tenant boundary)
  ├── Workspace Members (users in this workspace)
  ├── Standard Objects
  │     ├── Person ──── NoteTarget, TaskTarget, Favorite
  │     ├── Company ─── NoteTarget, TaskTarget, Favorite
  │     ├── Opportunity ── Pipeline Stages
  │     ├── Note ─────── NoteTarget (polymorphic)
  │     ├── Task ─────── TaskTarget (polymorphic)
  │     ├── Attachment
  │     ├── Dashboard ── Widgets
  │     ├── Workflow ─── Triggers, Actions
  │     ├── Connected Account ── Messages, Calendar Events
  │     ├── Message ──── Message Participants
  │     ├── Calendar Event ── Calendar Event Participants
  │     └── Blocklist
  ├── Custom Objects (user-defined at runtime)
  └── Views (saved filter/sort/group per object)
```

## Cross-References

- @SCHEMA.md — Detailed field definitions, types, relations
- @API.md — How terms map to GraphQL types and REST endpoints
- @UI.md — How terms appear in the interface
- @ARCHITECTURE.md — How terms map to code modules
