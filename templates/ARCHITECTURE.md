---
devmd: architecture
version: 0.1.0

structure:
  type: ""                       # monorepo | polyrepo | monolith | microservices
  package_manager: ""            # npm | pnpm | cargo | ...
  packages: []                   # list of package/module names

layers:
  - name: ""
    responsibility: ""
    allowed_imports: []           # which layers this can import
    forbidden_imports: []

dependency_rules:
  direction: ""                  # inward | downward
  rule: ""                       # e.g. "domain never imports infra"

tech_stack:
  language: ""
  framework: ""
  runtime: ""
---

# ARCHITECTURE.md

> System structure, layers, dependency rules, and architectural decisions.

## Overview

<!-- High-level system diagram description. Reference @PRODUCT.md#solution. -->

## Layers

<!-- Expand frontmatter layers. Show allowed dependency direction. -->

| Layer | Responsibility | May Import | Must Not Import |
|-------|---------------|------------|-----------------|
|       |               |            |                 |

## Package Structure

<!-- Directory tree or monorepo workspace map. Reference @INFRA.md#compute for deploy units. -->

## Key Decisions (ADRs)

<!-- Architecture Decision Records. One per decision. -->

### ADR-001: [Title]

- **Status:** proposed | accepted | deprecated
- **Context:**
- **Decision:**
- **Consequences:**

## Cross-References

- Data models: @SCHEMA.md
- API surface: @API.md
- Infrastructure: @INFRA.md
- Error handling: @ERRORS.md
