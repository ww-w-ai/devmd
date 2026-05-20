---
devmd: flows
version: 0.1.0

flows:
  - id: ""
    name: ""
    trigger: ""                  # e.g. "user clicks Sign Up"
    actor: ""                    # user | system | cron | webhook
    ux:
      - step: 1
        screen: ""               # ref @SCREENS.md#screen-id
        action: ""
    data:
      - step: 1
        endpoint: ""             # ref @API.md#endpoint
        payload: ""
    error_paths:
      - at_step: 0
        error: ""                # ref @ERRORS.md#code
        recovery: ""             # retry | redirect | message
    success_state: ""
---

# FLOWS.md

> User flows with paired UX and data steps, error paths, and success states.

## Flow List

<!-- One subsection per flow. Pair UX steps with data steps. -->

### [Flow Name]

- **ID:**
- **Trigger:**
- **Actor:**

| Step | UX (screen + action) | Data (endpoint + payload) |
|------|---------------------|--------------------------|
| 1    |                     |                          |
| 2    |                     |                          |
| 3    |                     |                          |

**Error Paths:**

| At Step | Error | Recovery |
|---------|-------|----------|
|         |       |          |

**Success State:**

## Flow Diagram

<!-- ASCII or mermaid diagram showing flow relationships. -->

## Cross-References

- Screens: @SCREENS.md
- API: @API.md
- Errors: @ERRORS.md
