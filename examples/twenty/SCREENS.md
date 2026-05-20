---
devmd: screens
version: "1.0"
project: Twenty CRM

design_system: "@DESIGN.md"
ui_architecture: "@UI.md"

themes: [light, dark]
responsive: false
note: >
  Twenty is a desktop-first SPA. No mobile-responsive layouts.
  Minimum viewport: 1024px.

screens:
  - id: record-list
    path: "/objects/{objectNamePlural}"
    layouts: [table, kanban, calendar]
    states: [default, empty, loading, filtered, grouped, bulk-selected]
  - id: record-detail
    path: "/objects/{objectNamePlural}/{recordId}"
    panels: [main, right-panel]
    states: [default, loading, editing, empty-timeline]
  - id: settings
    path: "/settings/{section}"
    sections: [workspace, members, roles, data-model, integrations, accounts, ai]
    states: [default, loading, editing, confirmation-dialog]
  - id: command-menu
    type: overlay
    trigger: "Cmd+K / Ctrl+K"
    states: [search, navigation, actions, recent]
  - id: dashboard
    path: "/dashboard"
    features: [drag-resize, chart-types, date-range-selector]
    states: [default, empty, editing, loading]
---

# Twenty CRM — Screen Reference

Visual reference for key screens. All screens use twenty-ui components and theme tokens defined in @DESIGN.md. Layout structure defined in @UI.md#app-layout.

## 1. Record List View

The primary data view. Users spend most time here. See @UI.md#page-types.

### Layout

```
┌──────────────────────────────────────────────────────────┐
│ Sidebar (220px)  │  Main Content                          │
│                  │                                        │
│ ┌──────────────┐ │  ┌────────────────────────────────────┐│
│ │ Navigation   │ │  │ Top Bar                            ││
│ │  People    ● │ │  │ ┌──────┐ ┌──────┐ ┌────┐ ┌─────┐ ││
│ │  Companies   │ │  │ │Filter│ │ Sort │ │View│ │+ New│ ││
│ │  Opport.     │ │  │ └──────┘ └──────┘ └────┘ └─────┘ ││
│ │  [Custom]    │ │  └────────────────────────────────────┘│
│ │              │ │                                        │
│ │ ──────────── │ │  ┌────────────────────────────────────┐│
│ │ Favorites    │ │  │ Data Table                         ││
│ │  ★ Jane S.   │ │  │ ┌────┬──────┬─────────┬──────────┐││
│ │  ★ Acme Corp │ │  │ │ ☐  │ Name │ Company │ Job Title│││
│ │              │ │  │ ├────┼──────┼─────────┼──────────┤││
│ │ ──────────── │ │  │ │ ☐  │ Jane │ Acme    │ CTO      │││
│ │ Views        │ │  │ │ ☐  │ John │ Widget  │ VP Sales │││
│ │  All People  │ │  │ │ ☐  │ Sara │ TechCo  │ Engineer │││
│ │  My Contacts │ │  │ │ ...│ ...  │ ...     │ ...      │││
│ │  + New View  │ │  │ └────┴──────┴─────────┴──────────┘││
│ └──────────────┘ │  │                        Page 1 of 5 ││
│                  │  └────────────────────────────────────┘│
└──────────────────────────────────────────────────────────┘
```

### Table Layout Features

| Feature | Interaction | Data |
|---|---|---|
| Column resize | Drag column borders | Width persisted in View metadata |
| Column reorder | Drag column header | Order persisted in View metadata |
| Inline edit | Click cell → edit in place | Auto-save on blur via GraphQL mutation |
| Sorting | Click column header | Multi-column sort with direction |
| Filtering | Filter button → field/operator/value | Filter groups with AND/OR logic |
| Grouping | Group button → select field | Collapsible groups by enum/relation |
| Bulk actions | Checkbox selection → action bar | Delete, export, merge, update field |
| Export | Bulk action → Export CSV | Server-side CSV generation |
| Pagination | Scroll or page controls | Cursor-based (first/after). See @API.md |

### Kanban Layout

Used primarily for Opportunities. See @GLOSSARY.md#pipeline.

```
┌──────────┬──────────┬──────────┬──────────┬──────────┐
│ Qualif.  │ Meeting  │ Proposal │ Negotiat.│ Won/Lost │
│ (3)      │ (2)      │ (4)      │ (1)      │ (5)      │
├──────────┼──────────┼──────────┼──────────┼──────────┤
│┌────────┐│┌────────┐│┌────────┐│┌────────┐│┌────────┐│
││Acme    │││Widget  │││TechCo  │││BigCorp │││Closed  ││
││$50,000 │││$30,000 │││$80,000 │││$200,000│││deals   ││
││60%     │││40%     │││75%     │││90%     │││here    ││
│└────────┘│└────────┘│└────────┘│└────────┘│└────────┘│
│┌────────┐│┌────────┐│┌────────┐│          │          │
││StartupX│││DataInc │││CloudCo ││          │          │
││$15,000 │││$25,000 │││$45,000 ││          │          │
│└────────┘│└────────┘│└────────┘│          │          │
│          │          │┌────────┐│          │          │
│          │          ││MedTech ││          │          │
│          │          ││$120,000││          │          │
│          │          │└────────┘│          │          │
└──────────┴──────────┴──────────┴──────────┴──────────┘

Drag cards between columns to change stage.
Stage change triggers workflow execution. See @FLOWS.md#workflow-execution-flow.
```

### States

**Default**: Table populated with records, filters/sorts applied per saved View.

**Empty**: No records exist for this object type.
```
┌────────────────────────────────────────────┐
│                                            │
│          (illustration: empty inbox)       │
│                                            │
│     No People yet                          │
│     Create your first person record        │
│     to get started.                        │
│                                            │
│         [ + Create Person ]                │
│                                            │
└────────────────────────────────────────────┘
```

**Loading**: Skeleton rows with shimmer animation (Framer Motion). Table header visible immediately; body rows show gray placeholder bars.

**Filtered (no results)**: View has active filters but zero matches.
```
┌────────────────────────────────────────────┐
│  Active filters: Company = "Acme" ✕        │
│                                            │
│     No results match your filters          │
│     Try removing some filters or           │
│     broadening your search.                │
│                                            │
│         [ Clear Filters ]                  │
│                                            │
└────────────────────────────────────────────┘
```

**Bulk Selected**: Checkbox column active. Action bar appears above table.
```
┌────────────────────────────────────────────┐
│  3 selected   [ Delete ] [ Export ] [More] │
│────────────────────────────────────────────│
│  ☑  Jane Smith    Acme Corp    CTO         │
│  ☑  John Doe      Widget Inc   VP Sales    │
│  ☑  Sara Lee      TechCo       Engineer    │
│  ☐  ...                                    │
└────────────────────────────────────────────┘
```

---

## 2. Record Detail

Full detail view for a single record. See @UI.md#page-types.

### Layout

```
┌──────────────────────────────────────────────────────────────────────┐
│ Sidebar │  Record Detail                           │  Right Panel    │
│         │                                          │  (400px)        │
│         │  ┌──────────────────────────────────────┐│                 │
│         │  │ Header                               ││  ┌───────────┐ │
│         │  │ ┌────┐  Jane Smith                   ││  │ Relations │ │
│         │  │ │ AV │  CTO at Acme Corp             ││  │           │ │
│         │  │ └────┘  jane@acme.com                ││  │ Company:  │ │
│         │  │         +1 555-0123                   ││  │ Acme Corp │ │
│         │  │                                      ││  │           │ │
│         │  │  ★ Favorite  ✎ Edit  ⋮ More          ││  │ Opps: (2) │ │
│         │  └──────────────────────────────────────┘│  │ - Deal A  │ │
│         │                                          │  │ - Deal B  │ │
│         │  ┌──────────────────────────────────────┐│  │           │ │
│         │  │ Fields Grid                          ││  │ Tasks: (1)│ │
│         │  │                                      ││  │ - Follow  │ │
│         │  │  City: San Francisco                 ││  │   up call │ │
│         │  │  LinkedIn: linkedin.com/in/jane      ││  │           │ │
│         │  │  Created: May 1, 2026                ││  └───────────┘ │
│         │  │  Created by: You (via API)           ││                 │
│         │  └──────────────────────────────────────┘│  ┌───────────┐ │
│         │                                          │  │ Notes     │ │
│         │  ┌──────────────────────────────────────┐│  │           │ │
│         │  │ Timeline (Activity)                  ││  │ Meeting   │ │
│         │  │                                      ││  │ notes     │ │
│         │  │ May 13 ── Email from Jane            ││  │ from...   │ │
│         │  │          "Re: Partnership proposal"  ││  │           │ │
│         │  │ May 12 ── Task completed             ││  └───────────┘ │
│         │  │          "Send proposal"             ││                 │
│         │  │ May 10 ── Note added                 ││                 │
│         │  │          "Discussed pricing..."      ││                 │
│         │  │ May 8  ── Record created             ││                 │
│         │  │          by You                      ││                 │
│         │  └──────────────────────────────────────┘│                 │
└──────────────────────────────────────────────────────────────────────┘
```

### Interactions

- **Header fields**: Click to edit inline. Avatar auto-loaded from Gravatar or uploaded.
- **Fields grid**: Click any value to edit. Composite fields (Address, FullName) expand to sub-fields. See @SCHEMA.md#composite-fields.
- **Timeline**: Reverse chronological. Filterable by type (emails, notes, tasks, events, changes). Infinite scroll.
- **Right panel**: Collapsible. Shows related records, notes preview, attachments. Click relation to navigate.
- **Rich text editor**: Notes and task body use TipTap/BlockNote. Supports markdown, mentions (@Person), embeds.

### States

**Loading**: Header skeleton + timeline skeleton. Right panel shows placeholder cards.

**Empty Timeline**: Record exists but no activity yet.
```
No activity yet. Add a note, create a task,
or connect an email account to see emails here.
[ + Add Note ]  [ + Create Task ]
```

**Editing (field)**: Clicked field shows input control. Other fields remain read-only. Auto-save on blur.

---

## 3. Settings Screens

Workspace configuration. Requires appropriate permissions. See @SECURITY.md#rbac.

### Settings Navigation

```
Settings
├── General           # Workspace name, logo, subdomain
├── Members           # Invite, manage roles
├── Roles             # Custom role definitions
├── Data Model        # Objects, fields, relations
├── Integrations      # Connected services
├── Accounts          # Connected email/calendar
├── AI                # AI provider, model selection
├── Billing           # Subscription, usage
└── Security          # Auth providers, 2FA, SSO
```

### Data Model Screen

Where custom objects and fields are managed. See @FLOWS.md#create-custom-object-flow.

```
┌──────────────────────────────────────────────────────────┐
│ Settings > Data Model                                     │
│                                                          │
│ Standard Objects (cannot delete)                         │
│ ┌──────────┬──────────┬────────────┬──────────────────┐  │
│ │ Icon     │ Name     │ Fields     │ Records          │  │
│ ├──────────┼──────────┼────────────┼──────────────────┤  │
│ │ 👤       │ Person   │ 14 fields  │ 1,234 records    │  │
│ │ 🏢       │ Company  │ 12 fields  │ 456 records      │  │
│ │ 💰       │ Opport.  │ 10 fields  │ 89 records       │  │
│ │ ...      │ ...      │ ...        │ ...              │  │
│ └──────────┴──────────┴────────────┴──────────────────┘  │
│                                                          │
│ Custom Objects                               [ + New ]   │
│ ┌──────────┬──────────┬────────────┬──────────────────┐  │
│ │ 📁       │ Project  │ 8 fields   │ 23 records       │  │
│ │ 🎯       │ Campaign │ 6 fields   │ 12 records       │  │
│ └──────────┴──────────┴────────────┴──────────────────┘  │
│                                                          │
│ Click object to view/edit fields and relations           │
└──────────────────────────────────────────────────────────┘
```

### Members Screen

```
┌──────────────────────────────────────────────────────────┐
│ Settings > Members                        [ Invite + ]   │
│                                                          │
│ ┌────────┬───────────────────┬──────────┬──────────────┐ │
│ │ Avatar │ Name / Email      │ Role     │ Status       │ │
│ ├────────┼───────────────────┼──────────┼──────────────┤ │
│ │ [JD]   │ Jane Doe          │ Owner    │ Active       │ │
│ │        │ jane@acme.com     │          │              │ │
│ │ [JS]   │ John Smith        │ Admin    │ Active       │ │
│ │        │ john@acme.com     │          │              │ │
│ │ [--]   │ sara@acme.com     │ Member   │ Invited      │ │
│ └────────┴───────────────────┴──────────┴──────────────┘ │
└──────────────────────────────────────────────────────────┘
```

### States

**Loading**: Section skeleton with card placeholders.

**Editing**: Inline editing for most fields. Modal dialogs for destructive actions (delete object, remove member).

**Confirmation Dialog**: Used for destructive actions.
```
┌──────────────────────────────────────┐
│  Delete "Project" object?            │
│                                      │
│  This will permanently delete the    │
│  object and all 23 records.          │
│  This action cannot be undone.       │
│                                      │
│      [ Cancel ]  [ Delete ]          │
└──────────────────────────────────────┘
```

---

## 4. Command Menu

Keyboard-driven navigation and actions. See @UI.md#app-layout.

### Layout

```
┌────────────────────────────────────────────────┐
│  🔍 Search people, companies, or type a command │
│                                                │
│  Recent                                        │
│  ┌────────────────────────────────────────────┐│
│  │  👤 Jane Smith — Acme Corp                 ││
│  │  🏢 Widget Inc — widgetinc.com             ││
│  │  💰 Enterprise Deal — $200,000             ││
│  └────────────────────────────────────────────┘│
│                                                │
│  Quick Actions                                 │
│  ┌────────────────────────────────────────────┐│
│  │  + Create Person                    Ctrl+N ││
│  │  + Create Company                          ││
│  │  ⚙ Go to Settings                  Ctrl+, ││
│  │  🔍 Search all records              Ctrl+K ││
│  └────────────────────────────────────────────┘│
└────────────────────────────────────────────────┘
```

### Search Mode

```
┌────────────────────────────────────────────────┐
│  🔍 jane                                       │
│                                                │
│  People (3)                                    │
│  ┌────────────────────────────────────────────┐│
│  │  👤 Jane Smith — CTO, Acme Corp           ││
│  │  👤 Jane Doe — VP Sales, TechCo           ││
│  │  👤 Janet Lee — Engineer, DataInc          ││
│  └────────────────────────────────────────────┘│
│                                                │
│  Companies (1)                                 │
│  ┌────────────────────────────────────────────┐│
│  │  🏢 Jane's Bakery LLC                     ││
│  └────────────────────────────────────────────┘│
│                                                │
│  Press Enter to open · Esc to close            │
└────────────────────────────────────────────────┘
```

### Interactions

- **Trigger**: `Cmd+K` (Mac) / `Ctrl+K` (Windows/Linux)
- **Navigation**: Arrow keys to select, Enter to open, Esc to close
- **Search**: Fuzzy search across all object types simultaneously
- **Actions**: Type `/` to see command list (create, navigate, settings)
- **AI**: Type `@` to invoke AI assistant (if AI enabled). See @AGENTS.md

### States

**Recent**: Default state showing recently viewed records.
**Search**: Active search query with grouped results.
**Actions**: Command list (triggered by `/` prefix).
**Empty search**: No results found for query.

---

## 5. Dashboard

Configurable analytics display. See @UI.md#page-types.

### Layout

```
┌──────────────────────────────────────────────────────────┐
│ Dashboard                          [ Edit ] [ Date ▾ ]   │
│                                                          │
│ ┌──────────────────────┐ ┌──────────────────────────────┐│
│ │ Pipeline Value       │ │ Deals by Stage               ││
│ │                      │ │                              ││
│ │    $1.2M             │ │  ████████ Won: 45            ││
│ │    total pipeline    │ │  ██████ Proposal: 32         ││
│ │                      │ │  ████ Meeting: 18            ││
│ │    ▲ 15% vs last mo  │ │  ██ Qualification: 8        ││
│ └──────────────────────┘ └──────────────────────────────┘│
│                                                          │
│ ┌──────────────────────────────────┐ ┌──────────────────┐│
│ │ New Contacts This Month          │ │ Tasks Due Today  ││
│ │                                  │ │                  ││
│ │  ┌─┐                            │ │ ☐ Call Jane      ││
│ │  │ │ ┌─┐                        │ │ ☐ Send proposal  ││
│ │  │ │ │ │ ┌─┐ ┌─┐               │ │ ☑ Update CRM     ││
│ │  │ │ │ │ │ │ │ │ ┌─┐           │ │                  ││
│ │  W1  W2  W3  W4  W5            │ │ 2 remaining      ││
│ └──────────────────────────────────┘ └──────────────────┘│
└──────────────────────────────────────────────────────────┘
```

### Widget Types

| Widget | Chart Library | Data Source |
|---|---|---|
| KPI Card | Custom | Aggregate query on any object |
| Bar Chart | Nivo | Group-by query (e.g., deals by stage) |
| Line Chart | Nivo | Time-series query (e.g., contacts per week) |
| Pie Chart | Nivo | Distribution query |
| Table | React Data Grid | Filtered record list |
| Task List | Custom | Tasks filtered by assignee + due date |

### Edit Mode

```
┌──────────────────────────────────────────────────────────┐
│ Dashboard (Editing)                   [ Done ] [ Cancel ] │
│                                                          │
│ ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐ ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐│
│ │ Pipeline Value  [⚙] │ │ Deals by Stage          [⚙] ││
│ │ (drag to move)      │ │ (drag corners to resize)     ││
│ └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘ └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘│
│                                                          │
│              [ + Add Widget ]                            │
│                                                          │
└──────────────────────────────────────────────────────────┘

Dashed borders indicate draggable/resizable widgets.
Grid snapping ensures alignment.
```

### States

**Default**: Dashboard with saved widget layout, auto-refreshing data.

**Empty**: New dashboard with no widgets.
```
┌────────────────────────────────────┐
│  Your dashboard is empty           │
│  Add widgets to track your         │
│  pipeline, contacts, and tasks.    │
│                                    │
│       [ + Add First Widget ]       │
└────────────────────────────────────┘
```

**Loading**: Widget cards with skeleton loaders. Layout grid visible immediately.

**Dark Mode**: All screens support dark theme via CSS custom properties. See @DESIGN.md#theme-switching.

---

## Dark Mode

All screens support light and dark themes. Theme toggle in Settings > General or user avatar dropdown. See @DESIGN.md#theme-switching.

Key differences in dark mode:
- Background: `#141414` (primary), `#1C1C1C` (secondary)
- Text: `#F2F2F2` (primary), `#999999` (secondary)
- Accent: `#6B93FF` (slightly lighter than light theme `#4D77FF`)
- Borders: `#333333`
- Cards: elevated surfaces use `#1C1C1C` with subtle border

No functional differences between themes. All interactions, states, and layouts are identical.

## Cross-References

- @DESIGN.md — Theme tokens, Linaria styling, twenty-ui components
- @UI.md — Page types, layouts, state management, routing
- @FLOWS.md — User journeys that traverse these screens
- @GLOSSARY.md — Domain terms (Person, Company, Opportunity, View, etc.)
- @SCHEMA.md — Data model that drives what fields appear on screens
- @SECURITY.md — Which settings screens are visible per role
