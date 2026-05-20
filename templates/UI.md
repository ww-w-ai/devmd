---
devmd: ui
version: 0.1.0

app:
  type: ""                       # spa | mpa | dashboard | mobile | hybrid
  framework: ""                  # react | vue | svelte | native | ...

navigation:
  type: ""                       # sidebar | top-nav | bottom-tab | hamburger
  items:
    - label: ""
      path: ""
      icon: ""
      auth: ""                   # public | authenticated | role:admin

layouts:
  - name: ""
    regions: []                  # e.g. [header, sidebar, main, footer]
    responsive: ""               # stack | hide-sidebar | ...

pages:
  - path: ""
    layout: ""
    sections:
      - type: ""                 # card-feed | table | form | hero | ...
        data_source: ""          # @API.md#endpoint or static
    auth: ""

patterns:
  loading: ""                    # skeleton | spinner | shimmer
  empty: ""                      # illustration | cta | message
  error: ""                      # inline | toast | page

transitions:
  page: ""                       # fade | slide | none
  component: ""                  # expand | fade | none
---

# UI.md

> Frontend structure, navigation, layouts, pages, and interaction patterns.

## App Type

<!-- SPA, MPA, dashboard, etc. Reference @ARCHITECTURE.md#tech-stack. -->

## Navigation

<!-- Nav structure and items. Reference @SECURITY.md#rbac for auth-gated items. -->

## Layouts

<!-- Layout templates with regions. Reference @DESIGN.md for visual tokens. -->

## Pages

<!-- One subsection per page. Use type abstractions (card-feed, table, form). -->

### [Page Name]

- **Path:** `/`
- **Layout:**
- **Sections:** `type: card-feed` → @API.md#endpoint
- **Auth:**

## Interaction Patterns

<!-- Loading, empty, error states. Reference @ERRORS.md for error display. -->

## Forms

<!-- Validation rules, submit behavior. Reference @API.md#endpoints for targets. -->

## Responsive Strategy

<!-- Breakpoints, layout shifts. Reference @DESIGN.md for spacing tokens. -->
