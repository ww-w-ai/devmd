---
devmd: lifecycle
version: "1.0"
project: Twenty CRM

state_machines:
  - name: workspace-provisioning
    states: [creating, seeding, ready, failed]
    initial: creating
    terminal: [ready, failed]
  - name: metadata-migration
    states: [pending, applying, applied, failed, rollback]
    initial: pending
    terminal: [applied, failed]
  - name: record-soft-delete
    states: [active, deleted, purged]
    initial: active
    terminal: [purged]
  - name: email-sync
    states: [idle, syncing, synced, error, auth-failed]
    initial: idle
    terminal: []
  - name: workflow-execution
    states: [triggered, evaluating, executing, completed, partial, failed]
    initial: triggered
    terminal: [completed, partial, failed]
  - name: connected-account
    states: [connecting, connected, syncing, auth-failed, disconnected]
    initial: connecting
    terminal: [disconnected]

devmd_lifecycle:
  phases: [concept, scaffold, implement, validate, deploy, operate]
  note: >
    Maps which DevMD files are primary inputs at each development phase.
---

# Twenty CRM — Lifecycle & State Machines

State transitions for key domain objects and processes. Each state machine documents valid states, transitions, guards, and side effects.

## 1. Workspace Provisioning

See @FLOWS.md#workspace-provisioning-flow for the full end-to-end flow.

### State Machine

```
                    ┌─────────────────────────┐
                    │                         │
                    ▼                         │
  ┌──────────┐  success  ┌─────────┐  success  ┌───────┐
  │ CREATING ├──────────►│ SEEDING ├──────────►│ READY │
  └────┬─────┘           └────┬────┘           └───────┘
       │                      │
       │ failure              │ failure
       │                      │
       ▼                      ▼
  ┌──────────┐           ┌──────────┐
  │  FAILED  │◄──────────│  FAILED  │
  └────┬─────┘           └──────────┘
       │
       │ retry (admin action)
       │
       ▼
  ┌──────────┐
  │ CREATING │  (restart from scratch)
  └──────────┘
```

### State Definitions

| State | Description | Entry Condition | Side Effects |
|---|---|---|---|
| CREATING | PostgreSQL schema being created | Signup completed, user verified | `CREATE SCHEMA workspace_{id}` |
| SEEDING | Standard objects, fields, relations, views being created | Schema created successfully | Metadata records created, tables created |
| READY | Workspace fully operational | All seeding steps completed | GraphQL schema generated, tokens issued |
| FAILED | Provisioning error at any step | Schema creation or seeding error | Schema dropped (if CREATING failed), admin alerted |

### Guards and Invariants

- A workspace must not serve API requests until state = READY.
- FAILED workspaces can be retried by admin via CLI. See @CLAUDE.md#development-workflow.
- Schema creation is idempotent — retry drops existing schema first.
- Seeding is NOT idempotent — partial seeding requires full re-seeding after schema drop.

---

## 2. Metadata Migration

Applies when the Twenty platform is upgraded and schema changes affect all workspaces. See @SCHEMA.md#multi-tenant.

### State Machine

```
  ┌─────────┐  start    ┌──────────┐  success  ┌─────────┐
  │ PENDING ├──────────►│ APPLYING ├──────────►│ APPLIED │
  └─────────┘           └────┬─────┘           └─────────┘
                              │
                              │ failure
                              │
                              ▼
                         ┌──────────┐  manual    ┌──────────┐
                         │  FAILED  ├───────────►│ ROLLBACK │
                         └──────────┘            └──────────┘
```

### Migration Levels

Two migration tracks run independently:

**Public schema migrations** (core platform tables):
- Standard TypeORM migrations versioned in code.
- Applied at server startup via CLI: `nx run twenty-server:migrate`.
- Affect all workspaces simultaneously (shared tables).

**Workspace schema migrations** (per-tenant):
- Metadata-driven. Applied dynamically when a workspace connects after upgrade.
- Each workspace tracks its own migration version.
- Migrate independently — failure in one workspace does not block others.

### State Definitions

| State | Description | Entry Condition |
|---|---|---|
| PENDING | New migration detected, not yet applied | Server detects version mismatch |
| APPLYING | Migration SQL executing | Migration runner started |
| APPLIED | Migration completed successfully | All SQL statements executed |
| FAILED | Migration error (constraint violation, timeout) | SQL error during APPLYING |
| ROLLBACK | Migration reversed to previous state | Admin triggers manual rollback |

### Guards

- Migrations run inside a transaction. Failure triggers automatic `ROLLBACK`.
- Workspace migrations are per-workspace. One workspace failing does not affect others.
- Public schema migrations block server startup until completed.
- Backup is mandatory before applying public schema migrations. See @INFRA.md#backup-strategy.

---

## 3. Record Soft-Delete

All workspace records use soft delete. See @GLOSSARY.md#record-soft-delete, @FLOWS.md#record-crud-with-permissions.

### State Machine

```
  ┌────────┐  delete     ┌─────────┐  30 days    ┌────────┐
  │ ACTIVE ├────────────►│ DELETED ├─────────────►│ PURGED │
  └────────┘             └────┬────┘              └────────┘
                              │
                              │ restore
                              │
                              ▼
                         ┌────────┐
                         │ ACTIVE │
                         └────────┘
```

### State Definitions

| State | Description | DB Representation | Query Behavior |
|---|---|---|---|
| ACTIVE | Normal state | `deletedAt = NULL` | Included in all queries by default |
| DELETED | Soft-deleted (in Trash) | `deletedAt = timestamp` | Excluded from queries unless `withDeleted: true` |
| PURGED | Permanently removed | Row physically deleted | Cannot be recovered |

### Transitions

| Transition | Trigger | Guard | Side Effect |
|---|---|---|---|
| ACTIVE → DELETED | User clicks Delete, API `deleteOne` mutation | DELETE permission on object | `deletedAt` set to `now()`, event emitted |
| DELETED → ACTIVE | User clicks Restore in Trash, API `restore` endpoint | DELETE permission on object | `deletedAt` set to `NULL`, event emitted |
| DELETED → PURGED | Cron job (daily, records older than 30 days) | Automatic, no user action | Physical `DELETE FROM` SQL, no recovery possible |

### Cascade Behavior

When a record is soft-deleted, related records follow these rules:

| Relation Type | Behavior on Parent Delete |
|---|---|
| ONE_TO_MANY (children) | Children remain ACTIVE (orphaned, not cascaded) |
| MANY_TO_ONE (parent FK) | FK set to NULL (SET_NULL) or cascade based on relation config |
| MANY_TO_MANY (join) | Join table entries removed |
| NoteTarget / TaskTarget | Targets soft-deleted (notes/tasks become orphaned) |

### twenty-orm Default Filter

```typescript
// All workspace queries automatically filter out soft-deleted records
// twenty-orm injects: WHERE deletedAt IS NULL
// Unless explicitly requested:
const allIncludingDeleted = await repository.find({
  withDeleted: true,  // includes DELETED records
});

const onlyDeleted = await repository.find({
  where: { deletedAt: Not(IsNull()) },  // Trash view
});
```

---

## 4. Email Sync Lifecycle

Per-connected-account sync cycle. See @RUNTIME.md#email-calendar-sync, @FLOWS.md#email-sync-flow.

### State Machine

```
  ┌──────┐  cron trigger  ┌─────────┐  success   ┌────────┐
  │ IDLE ├───────────────►│ SYNCING ├────────────►│ SYNCED │
  └──────┘                └────┬────┘             └───┬────┘
                               │                      │
                               │ error                │ 5 min
                               │                      │ (next cron)
                               ▼                      │
                          ┌─────────┐                 │
                          │  ERROR  │                 │
                          └────┬────┘                 │
                               │                      │
                               ├── retryable ─────────┘ (back to IDLE)
                               │
                               │ auth failure
                               ▼
                          ┌─────────────┐
                          │ AUTH-FAILED │
                          └──────┬──────┘
                                 │
                                 │ user re-authenticates
                                 ▼
                            ┌──────┐
                            │ IDLE │
                            └──────┘
```

### State Definitions

| State | Description | Duration | User Visibility |
|---|---|---|---|
| IDLE | Waiting for next sync cycle | ~5 minutes between syncs | "Connected" badge |
| SYNCING | Actively fetching messages from IMAP | 10s - 5min depending on volume | "Syncing..." spinner |
| SYNCED | Sync completed successfully | Momentary (transitions back to IDLE) | "Last synced: just now" |
| ERROR | Transient error (network, rate limit) | Until next retry | "Sync error" warning |
| AUTH-FAILED | OAuth token expired and refresh failed | Until user re-authenticates | "Re-authentication required" alert |

### Connected Account State Machine

The ConnectedAccount record itself has a lifecycle:

```
CONNECTING → CONNECTED → SYNCING ⇄ IDLE
                │
                │ auth failure
                ▼
           AUTH-FAILED → (user re-auth) → CONNECTED
                │
                │ user disconnects
                ▼
           DISCONNECTED (record soft-deleted)
```

---

## 5. Workflow Execution Lifecycle

Per-execution lifecycle for user-defined automations. See @RUNTIME.md#workflow-engine, @FLOWS.md#workflow-execution-flow.

### State Machine

```
  ┌───────────┐  match    ┌────────────┐  pass     ┌───────────┐
  │ TRIGGERED ├──────────►│ EVALUATING ├──────────►│ EXECUTING │
  └───────────┘           └─────┬──────┘           └─────┬─────┘
                                │                        │
                                │ conditions             ├── all succeed
                                │ not met                │
                                ▼                        ▼
                           ┌─────────┐            ┌───────────┐
                           │ SKIPPED │            │ COMPLETED │
                           └─────────┘            └───────────┘
                                                       │
                                                       │ some fail
                                                       ▼
                                                  ┌─────────┐
                                                  │ PARTIAL │
                                                  └─────────┘
                                                       │
                                                       │ critical fail
                                                       ▼
                                                  ┌────────┐
                                                  │ FAILED │
                                                  └────────┘
```

### State Definitions

| State | Description | Stored In |
|---|---|---|
| TRIGGERED | Event matched a workflow trigger | Job enqueued |
| EVALUATING | Conditions being checked against record data | Worker processing |
| SKIPPED | Conditions evaluated to false | Execution log (not stored as failure) |
| EXECUTING | Actions running sequentially | Worker processing |
| COMPLETED | All actions succeeded | Workflow execution history |
| PARTIAL | Some actions succeeded, some failed | Workflow execution history |
| FAILED | Critical action failed, execution halted | Workflow execution history |

### Action-Level States

Each action within an execution has its own state:

```
PENDING → RUNNING → SUCCEEDED
                  → FAILED (with retry)
                      → RUNNING (retry attempt)
                      → FAILED (max retries exceeded)
                  → SKIPPED (condition branch not taken)
```

### Execution History

Stored in the Workflow's `statuses` JSON field:

```json
{
  "executions": [
    {
      "id": "exec-uuid-1",
      "triggeredAt": "2026-05-13T10:30:00Z",
      "completedAt": "2026-05-13T10:30:02Z",
      "status": "completed",
      "triggerRecord": { "id": "opp-uuid", "objectName": "opportunity" },
      "actions": [
        { "type": "create-task", "status": "succeeded", "duration": 45 },
        { "type": "call-webhook", "status": "succeeded", "duration": 120 }
      ]
    }
  ]
}
```

---

## 6. DevMD Lifecycle — Files Active per Phase

Which DevMD files are primary inputs at each development phase.

### Phase Map

```
CONCEPT           SCAFFOLD          IMPLEMENT         VALIDATE          DEPLOY            OPERATE
─────────         ────────          ─────────         ────────          ──────            ───────
PRODUCT.md  ──►   ARCHITECTURE.md   CLAUDE.md         TESTING.md        INFRA.md          OPERATIONS.md
GLOSSARY.md ──►   SCHEMA.md         API.md            ERRORS.md         DEVOPS.md         LOGGING.md
BRAND.md    ──►   UI.md             AGENTS.md         SECURITY.md       CONFIG.md         LIFECYCLE.md
                  DESIGN.md         HARNESS.md        FLOWS.md                            CHANGELOG.md
                  INFRA.md          RUNTIME.md        SCREENS.md
                                    SEO.md
```

### Phase Descriptions

| Phase | Primary DevMD Files | Purpose |
|---|---|---|
| **Concept** | PRODUCT.md, GLOSSARY.md, BRAND.md | Define what to build, domain language, brand voice |
| **Scaffold** | ARCHITECTURE.md, SCHEMA.md, UI.md, DESIGN.md, INFRA.md | Define structure, data model, layout, visual design, infrastructure |
| **Implement** | CLAUDE.md, API.md, AGENTS.md, HARNESS.md, RUNTIME.md, SEO.md | Code conventions, API design, AI integration, runtime behavior |
| **Validate** | TESTING.md, ERRORS.md, SECURITY.md, FLOWS.md, SCREENS.md | Test strategy, error handling, security audit, flow verification |
| **Deploy** | INFRA.md, DEVOPS.md, CONFIG.md | Infrastructure, CI/CD, environment configuration |
| **Operate** | OPERATIONS.md, LOGGING.md, LIFECYCLE.md, CHANGELOG.md | Monitoring, state management, change history |

### AI Agent Usage by Phase

| Phase | AI reads these files | AI produces |
|---|---|---|
| Concept | PRODUCT.md, GLOSSARY.md | Nothing (human-driven) |
| Scaffold | ARCHITECTURE.md, SCHEMA.md, UI.md, DESIGN.md | Project skeleton, empty modules, DB migrations |
| Implement | All scaffold files + CLAUDE.md, API.md | Application code, resolvers, components, tests |
| Validate | TESTING.md, FLOWS.md, ERRORS.md | Test files, error handlers, security fixes |
| Deploy | INFRA.md, CONFIG.md, DEVOPS.md | Docker configs, CI workflows, Helm values |
| Operate | OPERATIONS.md, LOGGING.md | Runbooks, dashboards, alerts |

## Cross-References

- @FLOWS.md — End-to-end flows that traverse these state machines
- @SCHEMA.md — Multi-tenant architecture, soft-delete, migration strategy
- @RUNTIME.md — Worker queues, email sync, workflow engine
- @SECURITY.md — Authentication states, token lifecycle
- @ERRORS.md — Error handling at state transitions
- @LOGGING.md — State transition logging and metrics
- @INFRA.md — Backup strategy before migrations
