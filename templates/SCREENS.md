---
devmd: screens
version: 0.1.0

screens:
  - id: ""
    name: ""
    path: ""
    auth: ""                     # public | authenticated | role:admin
    layout: ""                   # ref @UI.md#layouts
    states:
      - name: ""                 # default | loading | empty | error
        description: ""
    annotations: []              # design notes, edge cases
    responsive:
      mobile: ""                 # stack | hide | drawer | ...
      tablet: ""
      desktop: ""

theme_comparison:
  modes: []                      # e.g. [light, dark]
  screens_to_compare: []         # screen ids requiring both mode previews
---

# SCREENS.md

> Screen inventory with paths, auth, states, responsive behavior, and theme modes.

## Screen List

<!-- One subsection per screen. Reference @UI.md#pages for section types. -->

### [Screen Name]

- **ID:**
- **Path:**
- **Auth:**
- **Layout:** @UI.md#layout-name
- **Design:** @DESIGN.md

| State | Description | Screenshot |
|-------|-------------|------------|
| default | | |
| loading | | |
| empty   | | |
| error   | | |

**Responsive:** mobile: | tablet: | desktop:

**Annotations:**

## Theme Comparison

<!-- Which screens need light/dark preview. Reference @DESIGN.md for color tokens. -->

## Screen Map

<!-- Visual sitemap or tree showing screen hierarchy and navigation paths. -->
