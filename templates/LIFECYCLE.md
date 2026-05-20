---
devmd: lifecycle
version: 0.1.0

development_states:
  - state: planning
    active_specs:
      - PRODUCT.md
      - GLOSSARY.md
    actions: []                  # what happens in this state
    gates: []                    # conditions to exit
    exit: designing
    next: designing

  - state: designing
    active_specs:
      - ARCHITECTURE.md
      - SCHEMA.md
      - DESIGN.md
      - UI.md
    actions: []
    gates: []
    exit: implementing
    next: implementing

  - state: implementing
    active_specs:
      - API.md
      - ERRORS.md
      - LOGGING.md
    actions: []
    gates: []
    exit: testing
    next: testing

  - state: testing
    active_specs:
      - TESTING.md
      - SECURITY.md
    actions: []
    gates: []
    exit: deploying
    next: deploying

  - state: deploying
    active_specs:
      - DEVOPS.md
      - INFRA.md
      - CONFIG.md
    actions: []
    gates: []
    exit: operating
    next: operating

  - state: operating
    active_specs:
      - OPERATIONS.md
      - CHANGELOG.md
    actions: []
    gates: []
    exit: planning
    next: planning

agent_states:
  - state: ""
    allowed_actions: []
    transitions: []

transition_rules:
  - from: ""
    to: ""
    condition: ""                # gate that must pass
    auto: false                  # auto-transition or manual

conflict_detection:
  strategy: ""                   # lock | merge | notify
  concurrent_limit: 0
---

# LIFECYCLE.md

> Development states, agent states, transition rules, and conflict detection.

## Development States

<!-- Which specs are active in each phase. -->

| State | Active Specs | Gates | Next |
|-------|-------------|-------|------|
| planning | PRODUCT, GLOSSARY | | designing |
| designing | ARCHITECTURE, SCHEMA, DESIGN, UI | | implementing |
| implementing | API, ERRORS, LOGGING | | testing |
| testing | TESTING, SECURITY | | deploying |
| deploying | DEVOPS, INFRA, CONFIG | | operating |
| operating | OPERATIONS, CHANGELOG | | planning |

## Transition Rules

<!-- Conditions for moving between states. Reference @TESTING.md for test gates. -->

## Agent States

<!-- If agents manage the lifecycle. Reference @AGENTS.md and @HARNESS.md. -->

## Conflict Detection

<!-- How concurrent edits to specs are handled. -->

## Cross-References

- Agent definitions: @AGENTS.md
- Agent harness: @HARNESS.md
- Runtime: @RUNTIME.md
