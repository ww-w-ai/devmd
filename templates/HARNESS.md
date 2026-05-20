---
devmd: harness
version: 0.1.0

intelligence:
  primary: ""                    # model name or provider
  routing: ""                    # how tasks are routed to models
  fallback: ""                   # fallback model
  shadow: ""                     # shadow model for eval

knowledge:
  rag:
    enabled: false
    source: ""
    chunk_size: 0
    overlap: 0
  graph: ""                      # knowledge graph provider or none
  corpus: []                     # static document paths
  dedup: ""                      # deduplication strategy
  citation: ""                   # how sources are cited

tools:
  mcp: []                        # MCP server names
  apis: []                       # external API integrations
  browser: ""                    # browser tool config
  filesystem: ""                 # allowed paths or scope

guardrails:
  safety_levels: []              # e.g. [low, medium, high, critical]
  pace: ""                       # auto | confirm-each | batch
  content_filter: ""             # provider | custom | none
  confidence_threshold: 0        # minimum confidence to act
  approval_required: []          # actions requiring human approval
  daily_caps:
    actions: 0
    tokens: 0
    cost_usd: 0

memory:
  working: ""                    # how context is managed within session
  long_term: ""                  # persistence between sessions
  shared: ""                     # cross-agent memory
  session: ""                    # session storage strategy
  persona: ""                    # persistent persona data

team:
  roster: []                     # ref @AGENTS.md
  handoff: ""                    # sequential | parallel | event
  concurrency: 0                 # max parallel agents
  pdca_mapping: {}               # which agent handles which PDCA phase

lifecycle:
  trigger: ""                    # manual | schedule | webhook | event
  rounds: 0                      # max rounds per session
  timeout: ""                    # session timeout
  retry: ""                      # retry policy
  health: ""                     # health check endpoint
  shutdown: ""                   # graceful shutdown strategy

state_machines: []               # FSM definitions for complex workflows

observability:
  events: []                     # ref @LOGGING.md#audit-events
  dashboard: ""
  alerts: []

configuration:
  path: ""                       # config file path
  overrides: ""                  # how runtime overrides work

cost_control:
  budget_usd: 0                  # monthly budget
  alerts: []                     # cost alert thresholds
  optimization: ""               # caching, batching, routing

evaluation:
  metrics: []                    # quality metrics
  eval_frequency: ""
  baseline: ""

caching:
  prompt: ""                     # prompt caching strategy
  result: ""                     # result caching strategy
  ttl: ""
---

# HARNESS.md

> Agent execution harness — intelligence, knowledge, tools, guardrails, memory, team, and lifecycle.

## Intelligence

<!-- Model selection, routing, fallback. -->

## Knowledge

<!-- RAG, knowledge graph, corpus, citation rules. -->

## Tools

<!-- MCP servers, APIs, browser, filesystem access. -->

## Guardrails

<!-- Safety levels, pace, content filter, approval gates. Reference @SECURITY.md. -->

## Memory

<!-- Working, long-term, shared, session, persona memory. -->

## Team

<!-- Agent roster and coordination. Reference @AGENTS.md for agent details. -->

## Lifecycle

<!-- Trigger, rounds, timeout, retry, health, shutdown. Reference @RUNTIME.md. -->

## State Machines

<!-- FSM definitions for complex multi-step workflows. -->

## Observability

<!-- Event logging, dashboards, alerts. Reference @LOGGING.md. -->

## Cost Control

<!-- Budget, alerts, optimization. -->

## Evaluation

<!-- Quality metrics, eval frequency, baselines. Reference @TESTING.md. -->

## Caching

<!-- Prompt and result caching strategies. -->
