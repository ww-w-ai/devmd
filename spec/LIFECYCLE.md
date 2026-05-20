# LIFECYCLE.md Specification

> Version: 0.1.0-draft | Status: Draft | DevMD First

## Purpose

LIFECYCLE.md is the orchestrator that declares when each DevMD file is active. It defines development phases, agent states, transition rules, and conflict detection. Every other DevMD file describes **what**; LIFECYCLE.md describes **when**.

## Frontmatter Schema

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Lifecycle configuration name |
| `development` | `Development` | REQUIRED | Development phase state machine |
| `agent` | `map<string, AgentState>` | OPTIONAL | Agent lifecycle states |
| `transitions` | `Transitions` | OPTIONAL | Global transition rules and conflict detection |

### Development

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `states` | `map<string, DevState>` | REQUIRED | Development phase definitions. Min 2. |

### DevState

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `active_specs` | `string[]` | REQUIRED | DevMD files that are primary in this state. Min 1. |
| `actions` | `Action[]` | OPTIONAL | Available actions in this state |
| `entry` | `string` | OPTIONAL | Condition to enter this state |
| `exit` | `string` | OPTIONAL | Condition to leave this state |
| `gates` | `Gate[]` | OPTIONAL | Quality gates between states |
| `next` | `string[]` | REQUIRED | Valid next states. Min 1. |

**Action**: `{name: string, ref: @reference}` — both REQUIRED.
**Gate**: `{type: string, between: string[2], on_conflict: enum(block|warn|surface_to_user)}` — `type` and `between` REQUIRED.

### AgentState

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `load` | `@reference[]` | OPTIONAL | Files to load when entering this state |
| `enforce` | `@reference[]` | OPTIONAL | Files whose rules are enforced in this state |
| `next` | `string[]` | REQUIRED | Valid next states. Min 1. |

### Transitions

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `global_rules` | `string[]` | OPTIONAL | Rules that apply to all transitions |
| `conflict_detection` | `ConflictDetection` | OPTIONAL | Cross-file conflict checks |

### ConflictDetection

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `trigger` | `enum(on_state_change\|on_file_edit\|manual)` | OPTIONAL | When to run conflict checks |
| `checks` | `ConflictCheck[]` | OPTIONAL | Conflict check definitions |
| `on_conflict` | `enum(block\|warn\|surface_to_user)` | OPTIONAL | Default conflict resolution |

**ConflictCheck**: `{between: string[2], description: string}` — both REQUIRED.

## Body Sections

Sections MUST appear in this order when present.

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## Development Lifecycle` | REQUIRED | State descriptions, entry/exit criteria, phase rationale. |
| `## Agent Lifecycle` | OPTIONAL | Agent state descriptions and transitions. |
| `## Transition Rules` | REQUIRED | Global rules, gate descriptions, transition conditions. |
| `## Conflict Detection` | REQUIRED | Cross-file conflict check descriptions and resolution policies. |

## Cross-References

LIFECYCLE.md references ALL other DevMD files by design.

- `active_specs` entries MUST use `@FILE.md` syntax.
- `actions[].ref` MUST use `@FILE.md#section` syntax.
- `load` and `enforce` entries MUST use `@FILE.md` syntax.

## Validation Rules

1. Every state's `next[]` MUST reference states defined in `development.states`.
2. No orphan states: every state MUST be reachable from at least one other state, or be the initial state.
3. `active_specs` MUST only list DevMD files that exist in the project.
4. `development.states` MUST contain at least 2 states.
5. `ConflictCheck.between` MUST contain exactly 2 file references.
6. `Gate.between` MUST contain exactly 2 state names that exist in `development.states`.

## Conflict Detection

- `active_specs` in each state MUST only list files that exist in the project.
- If `agent` states reference files via `load` or `enforce`, those files MUST exist.
- State transitions MUST NOT create cycles without an explicit exit condition.
- `conflict_detection.checks` SHOULD cover all file pairs that share overlapping concerns.
