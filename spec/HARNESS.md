# HARNESS.md Specification

> Version: 0.1.0-draft | Status: Draft | DevMD First

## Purpose

HARNESS.md specifies the agent control plane — LLM configuration, RAG pipelines, tool access, guardrails, memory, team coordination, and lifecycle management. No prior standard exists for declarative agent infrastructure specification. AGENTS.md defines **what** an agent is; HARNESS.md defines **how** the agent operates.

## Frontmatter Schema

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Harness configuration name |
| `agents` | `map<string, Agent>` | REQUIRED | Agent definitions keyed by agent id |

A single-agent system MAY define agent fields at root level instead of under `agents`.

### Agent

Each agent contains up to 13 sections. Only `intelligence` and `lifecycle` are REQUIRED.

#### Section 1: Intelligence

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `intelligence.primary` | `PrimaryModel` | REQUIRED | Primary LLM configuration |
| `intelligence.routing` | `map<string, string>` | OPTIONAL | Purpose-to-model routing |
| `intelligence.fallback` | `Fallback[]` | OPTIONAL | Fallback model chain |
| `intelligence.shadow` | `Shadow` | OPTIONAL | Shadow model for A/B comparison |
| `intelligence.prompt_versioning` | `PromptVersioning` | OPTIONAL | Prompt version tracking |

**PrimaryModel**: `{model: string, temperature: number, max_tokens: number, system_prompt: string}` — all REQUIRED.
**Fallback**: `{provider: string, trigger: string}` — both REQUIRED.
**Shadow**: `{model: string, purpose: string}` — both REQUIRED.
**PromptVersioning**: `{strategy: enum(hash|semver|none)}` — REQUIRED.

#### Section 2: Knowledge

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `knowledge.rag` | `RAGSource[]` | OPTIONAL | Retrieval-augmented generation sources |
| `knowledge.graph` | `{provider: string, types: string[]}` | OPTIONAL | Knowledge graph |
| `knowledge.corpus` | `{path: string, format: string, rolling_window?: string}` | OPTIONAL | Document corpus |
| `knowledge.dedup` | `{strategy: enum(hash\|time-window\|none), window?: string}` | OPTIONAL | Content deduplication |
| `knowledge.citation` | `{enabled: boolean, format?: string}` | OPTIONAL | Citation configuration |

**RAGSource**: `{source: enum(vector_db|filesystem|api), provider: string, collection?: string, paths?: string[], top_k?: number, similarity?: number}`.

#### Section 3: Tools

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `tools.mcp_servers` | `MCPServer[]` | OPTIONAL | MCP server connections |
| `tools.apis` | `APITool[]` | OPTIONAL | Direct API integrations |
| `tools.browser` | `{provider: string, purpose: string}` | OPTIONAL | Browser automation |
| `tools.filesystem` | `{read: string[], write: string[], deny: string[]}` | OPTIONAL | File access rules |

**MCPServer**: `{name: string, transport: string, purpose: string, permissions?: string[]}`.
**APITool**: `{name: string, endpoint: url, methods: string[], auth?: string}`.

#### Section 4: Guardrails

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `guardrails.safety_levels` | `map<string, SafetyLevel>` | OPTIONAL | Tiered safety controls |
| `guardrails.pace` | `{min_interval: string, jitter: string, cooldown: string}` | OPTIONAL | Rate limiting |
| `guardrails.content_filter` | `{patterns: string[], action: enum(block\|flag\|redact)}` | OPTIONAL | Content filtering |
| `guardrails.confidence` | `{threshold: number, below_action: enum(block\|review)}` | OPTIONAL | Confidence gating |
| `guardrails.approval_gates` | `ApprovalGate[]` | OPTIONAL | Human-in-the-loop gates |
| `guardrails.daily_caps` | `map<string, number>` | OPTIONAL | Daily action limits |
| `guardrails.budget` | `{max_tokens_per_run: number, max_cost_per_run: number}` | OPTIONAL | Cost controls |

**SafetyLevel**: `{actions: string[], auto_execute: boolean}`.
**ApprovalGate**: `{before: string, approver: string, criteria: string}`.

Autonomous agents MUST have a `guardrails` section.

#### Section 5: Memory

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `memory.working` | `{format: string, buckets: string[]}` | OPTIONAL | Working memory |
| `memory.long_term` | `{provider: string, path: string, format: string}` | OPTIONAL | Persistent memory |
| `memory.shared` | `SharedMemory[]` | OPTIONAL | Cross-agent shared state |
| `memory.persona` | `{base: string, overlays: string[]}` | OPTIONAL | Persona memory |

**SharedMemory**: `{name: string, path: string, access: enum(read_only|read_write)}`.

#### Section 6: Team

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `team.reports_to` | `string` | OPTIONAL | Supervising agent id |
| `team.collaborates_with` | `string[]` | OPTIONAL | Peer agent ids |
| `team.delegates_to` | `string[]` | OPTIONAL | Subordinate agent ids |
| `team.concurrency` | `{parallel: string[], sequential: string[]}` | OPTIONAL | Execution ordering |
| `team.pdca_mapping` | `enum(research\|plan\|design\|do\|check\|act)` | OPTIONAL | PDCA phase responsibility |

#### Section 7: Lifecycle

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `lifecycle.trigger` | `{type: enum(scheduled\|event\|command\|webhook), config: any}` | REQUIRED | How the agent is activated |
| `lifecycle.timeout` | `string` | OPTIONAL | Maximum run duration |
| `lifecycle.retry` | `{max: number, backoff: string}` | OPTIONAL | Retry policy |
| `lifecycle.health_check` | `string` | OPTIONAL | Health check endpoint or command |

#### Sections 8-13: Extended

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `state_machines` | `map<string, {states: string[], transitions: any[]}>` | OPTIONAL | FSM definitions |
| `observability` | `{logging: any, metrics: any, reporting: any}` | OPTIONAL | Observability config |
| `configuration` | `{hierarchy: string[], secrets: any, feature_flags: any}` | OPTIONAL | Config management |
| `cost_control` | `{token_budget: any, model_routing: any, action_caps: any, rate_limit: any}` | OPTIONAL | Cost management |
| `evaluation` | `{quality_scoring: any, ab_testing: any, learning_loop: any, measurement: any}` | OPTIONAL | Quality evaluation |
| `caching` | `{prompt_cache: any, content_dedup: any, settings_cache: any}` | OPTIONAL | Caching strategies |

## Body Sections

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## Agents` | REQUIRED | Agent overview and responsibilities. |
| `## Intelligence` | REQUIRED | Model selection rationale. |
| `## Knowledge` | OPTIONAL | RAG pipeline and corpus details. |
| `## Tools` | OPTIONAL | Tool integration details. |
| `## Guardrails` | REQUIRED (autonomous) | Safety controls documentation. |
| `## Memory` | OPTIONAL | Memory architecture. |
| `## Team` | OPTIONAL | Multi-agent coordination. |
| `## Lifecycle` | REQUIRED | Trigger and retry documentation. |
| `## State Machines` | OPTIONAL | State transition diagrams. |
| `## Observability` | OPTIONAL | Logging and metrics setup. |
| `## Configuration` | OPTIONAL | Config hierarchy. |
| `## Cost Control` | OPTIONAL | Budget and rate limiting. |
| `## Evaluation` | OPTIONAL | Quality measurement. |
| `## Caching` | OPTIONAL | Cache strategies. |

## Cross-References

- MUST reference `@AGENTS.md` for agent persona definitions.
- SHOULD reference `@RUNTIME.md` for job execution infrastructure.
- SHOULD reference `@CONFIG.md` for environment variables and secrets.
- SHOULD reference `@SECURITY.md` for authentication and authorization.

## Validation Rules

1. Every agent with `lifecycle.trigger.type` = `scheduled` MUST have a `timeout`.
2. Autonomous agents (agents with `lifecycle.trigger.type` != `command`) MUST have a `guardrails` section.
3. `team.reports_to` and `team.delegates_to` MUST reference agent ids defined in `agents`.
4. `intelligence.primary` fields (`model`, `temperature`, `max_tokens`, `system_prompt`) are all REQUIRED.

## Conflict Detection

- Agent ids in `agents` MUST match agent names in `@AGENTS.md` if both files exist.
- `tools.mcp_servers` SHOULD NOT duplicate servers already declared in `@RUNTIME.md`.
- `guardrails.daily_caps` SHOULD be consistent with `cost_control.action_caps` if both are set.
