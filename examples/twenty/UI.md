---
devmd: ui
version: "1.0"
project: Twenty CRM

framework: "React 18"
bundler: "Vite 7 + SWC"
router: "react-router-dom 6.4"
state:
  global: Recoil
  local: Jotai
  server: "Apollo Client"
i18n:
  library: Lingui
  locales: "28+ (en, fr, de, es, ko, ja, zh, pt, it, nl, ...)"
rich_text: "TipTap + BlockNote"
code_editor: Monaco Editor
data_grid: React Data Grid
charts: Nivo
animation: Framer Motion

frontend_modules: "55+"

app_layout:
  type: sidebar-main
  regions:
    - name: sidebar
      width: 220px
      collapsible: true
      contents: [navigation, favorites, views, workspace-switcher]
    - name: main
      contents: [top-bar, content-area]
    - name: right-panel
      width: 400px
      collapsible: true
      contents: [record-detail, activity-timeline]
    - name: command-menu
      type: overlay
      trigger: "Cmd+K / Ctrl+K"
      contents: [search, navigation, actions]

page_types:
  - type: object-list
    pattern: "/objects/{objectNamePlural}"
    layouts: [table, kanban, calendar]
    features: [filtering, sorting, grouping, column-resize, inline-edit, bulk-actions, export]
  - type: record-detail
    pattern: "/objects/{objectNamePlural}/{recordId}"
    features: [field-edit, activity-timeline, relations, notes, tasks, attachments]
  - type: settings
    pattern: "/settings/{section}"
    sections: [workspace, members, roles, data-model, integrations, accounts, ai]
  - type: dashboard
    pattern: "/dashboard"
    features: [widgets, charts, drag-drop-layout]
  - type: auth
    pattern: "/auth/{action}"
    actions: [login, signup, verify-email, reset-password, sso-callback]
---

# Twenty CRM — UI Architecture

## Application Shell

```
┌─────────────────────────────────────────────────────────────────┐
│ ┌──────────┐ ┌────────────────────────────────┐ ┌────────────┐ │
│ │          │ │         Top Bar               │ │            │ │
│ │          │ │ ← Back │ Object Name │ + Add  │ │  Right     │ │
│ │ Sidebar  │ ├────────────────────────────────┤ │  Panel     │ │
│ │          │ │                                │ │            │ │
│ │ Search   │ │       Content Area             │ │ Record     │ │
│ │ ──────── │ │                                │ │ Detail     │ │
│ │ Favorites│ │  ┌─ Filters ──────────────┐   │ │            │ │
│ │ ──────── │ │  │ Company = Acme │ + Add │   │ │ Fields     │ │
│ │ People   │ │  └────────────────────────┘   │ │ Timeline   │ │
│ │ Companies│ │                                │ │ Notes      │ │
│ │ Opps     │ │  ┌─ Table View ───────────┐   │ │ Tasks      │ │
│ │ ──────── │ │  │ Name   │ Company │ Email│   │ │            │ │
│ │ Views    │ │  │ Alice  │ Acme    │ a@.. │   │ │            │ │
│ │  My View │ │  │ Bob    │ Beta    │ b@.. │   │ │            │ │
│ │  Team    │ │  │ Carol  │ Gamma   │ c@.. │   │ │            │ │
│ │ ──────── │ │  └────────────────────────┘   │ │            │ │
│ │ Custom   │ │                                │ │            │ │
│ │ Objects  │ │  Showing 1-20 of 156  ← 1 2 →│ │            │ │
│ │ Settings │ │                                │ │            │ │
│ └──────────┘ └────────────────────────────────┘ └────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## Routing

```typescript
// Route structure (react-router-dom 6.4)
<Routes>
  {/* Auth */}
  <Route path="/auth/login" element={<Login />} />
  <Route path="/auth/signup" element={<Signup />} />
  <Route path="/auth/verify-email" element={<VerifyEmail />} />
  <Route path="/auth/reset-password" element={<ResetPassword />} />
  <Route path="/auth/sso/callback" element={<SSOCallback />} />

  {/* Main App (requires auth) */}
  <Route element={<AppLayout />}>
    {/* Object list views */}
    <Route path="/objects/:objectNamePlural" element={<ObjectList />} />
    {/* Record detail */}
    <Route path="/objects/:objectNamePlural/:recordId" element={<RecordDetail />} />
    {/* Dashboard */}
    <Route path="/dashboard" element={<Dashboard />} />
    {/* Settings */}
    <Route path="/settings" element={<SettingsLayout />}>
      <Route path="workspace" element={<WorkspaceSettings />} />
      <Route path="members" element={<MemberSettings />} />
      <Route path="roles" element={<RoleSettings />} />
      <Route path="data-model" element={<DataModelSettings />} />
      <Route path="integrations" element={<IntegrationSettings />} />
      <Route path="accounts" element={<AccountSettings />} />
      <Route path="ai" element={<AISettings />} />
    </Route>
  </Route>
</Routes>
```

## State Management

Three-layer state architecture. See @ARCHITECTURE.md#adrs for the ADR.

### Recoil (Global State)

```typescript
// Workspace-level state
export const currentWorkspaceState = atom<Workspace>({
  key: 'currentWorkspace',
  default: null,
});

export const currentUserState = atom<User>({
  key: 'currentUser',
  default: null,
});

// Navigation state
export const navigationExpandedState = atom<boolean>({
  key: 'navigationExpanded',
  default: true,
});

// View state
export const currentViewIdState = atom<string | null>({
  key: 'currentViewId',
  default: null,
});

// Derived state
export const currentViewSelector = selector({
  key: 'currentView',
  get: ({ get }) => {
    const viewId = get(currentViewIdState);
    const views = get(viewsForCurrentObjectState);
    return views.find(v => v.id === viewId);
  },
});
```

### Jotai (Local/Component State)

```typescript
// Modal state (component-tree scoped)
export const personFormModalAtom = atom<{ open: boolean; personId?: string }>({
  open: false,
});

// Filter builder state (local to filter component tree)
export const filterBuilderAtom = atom<FilterGroup[]>([]);

// Inline edit state
export const inlineEditAtom = atom<{ recordId: string; fieldName: string } | null>(null);
```

### Apollo Client (Server State)

```typescript
// People query with filters, sorts, pagination
const { data, loading, fetchMore } = useQuery(FIND_PEOPLE, {
  variables: {
    filter: currentFilter,
    orderBy: currentSort,
    first: 20,
    after: cursor,
  },
});

// Optimistic update on person creation
const [createPerson] = useMutation(CREATE_ONE_PERSON, {
  optimisticResponse: {
    createOnePerson: {
      __typename: 'Person',
      id: tempId(),
      ...formData,
    },
  },
  update: (cache, { data }) => {
    // Update people list cache
  },
});
```

## Views

Views are saved configurations for displaying object records. See @GLOSSARY.md#view.

### View Types

| Type | Description | Use Case |
|---|---|---|
| Table | Spreadsheet-like grid with sortable columns | Default for most objects |
| Kanban | Card columns grouped by a field (e.g., pipeline stage) | Opportunities, tasks |
| Calendar | Monthly/weekly calendar layout | Calendar events, tasks with due dates |

### View Features

- **Filters** — Filter groups with AND/OR logic. Supports all field types and operators. See @API.md#filtering.
- **Sorts** — Multi-level sorting. Drag to reorder sort priority.
- **Groups** — Group records by a field value (e.g., group people by company).
- **Column configuration** — Show/hide fields, resize columns, reorder columns.
- **Saved views** — Save filter/sort/column config. Share with team or keep private.
- **Inline editing** — Click a cell to edit. Auto-saves on blur.
- **Bulk actions** — Select multiple records → delete, export, update field.

### Record Detail Page

```
┌──────────────────────────────────────────────┐
│  ← Back to People                            │
│  ┌──────┐                                    │
│  │Avatar│  Jane Smith                         │
│  └──────┘  CTO at Acme Corp                  │
│                                              │
│  ┌─ Fields ─────────────────────────────────┐│
│  │ Email     │ jane@acme.com        [edit]  ││
│  │ Phone     │ +1 555-0123          [edit]  ││
│  │ City      │ San Francisco        [edit]  ││
│  │ LinkedIn  │ linkedin.com/in/jane [edit]  ││
│  │ Company   │ Acme Corp            [link]  ││
│  │ + Add field                              ││
│  └──────────────────────────────────────────┘│
│                                              │
│  ┌─ Timeline ───────────────────────────────┐│
│  │ [Note] [Task] [Email] [Calendar] [All]   ││
│  │                                          ││
│  │ May 13 — Email: "Re: Partnership"        ││
│  │ May 12 — Note: "Discussed Q3 roadmap"    ││
│  │ May 10 — Task: "Send proposal" ✓         ││
│  │ May 8  — Created by John via API         ││
│  └──────────────────────────────────────────┘│
└──────────────────────────────────────────────┘
```

## Command Menu

Triggered by `Cmd+K` / `Ctrl+K`. Global search + navigation + actions.

```
┌─────────────────────────────────────┐
│ 🔍 Type to search...                │
│                                     │
│ Recent                              │
│   Jane Smith — Person               │
│   Acme Corp — Company               │
│                                     │
│ Navigation                          │
│   → People                          │
│   → Companies                       │
│   → Opportunities                   │
│   → Settings                        │
│                                     │
│ Actions                             │
│   + Create Person                   │
│   + Create Company                  │
│   + Create Opportunity              │
│   ⚙ Open Settings                   │
└─────────────────────────────────────┘
```

## Frontend Module Map

```
twenty-front/src/modules/
├── auth/                  # Login, signup, SSO, token management
├── people/                # Person list, detail, forms
├── companies/             # Company list, detail, forms
├── opportunities/         # Pipeline, kanban, deal detail
├── notes/                 # Rich text notes (TipTap + BlockNote)
├── tasks/                 # Task management, due dates
├── attachments/           # File upload, preview
├── views/                 # View configuration, filters, sorts
├── settings/              # All settings pages
│   ├── workspace/
│   ├── members/
│   ├── roles/
│   ├── data-model/        # Custom objects/fields UI
│   ├── integrations/
│   ├── accounts/
│   └── ai/
├── dashboard/             # Dashboard builder, widgets
├── workflow/              # Workflow builder UI
├── messaging/             # Email thread view
├── calendar/              # Calendar integration view
├── command-menu/          # Global command palette
├── navigation/            # Sidebar, breadcrumbs, routing
├── error-handler/         # Error boundaries, toast, Sentry
├── search/                # Global search across objects
├── favorites/             # Bookmarked records
├── data-grid/             # React Data Grid wrapper + custom cells
├── rich-text/             # TipTap + BlockNote editors
├── code-editor/           # Monaco Editor for JSON/code fields
├── charts/                # Nivo chart wrappers
├── i18n/                  # Lingui setup, locale management
├── theme/                 # Theme switching (light/dark)
├── workspace/             # Workspace selection, creation
├── connected-accounts/    # Email/calendar account linking
├── import-export/         # CSV import, data export
├── bulk-actions/          # Multi-select operations
├── keyboard-shortcuts/    # Global hotkey management
├── onboarding/            # New user onboarding flow
├── notification/          # In-app notifications
├── field-types/           # Field type renderers and editors
│   ├── text/
│   ├── number/
│   ├── date/
│   ├── enum/
│   ├── currency/
│   ├── full-name/
│   ├── address/
│   ├── link/
│   ├── emails/
│   ├── phones/
│   ├── rating/
│   ├── rich-text/
│   ├── json/
│   └── relation/
└── ... (55+ total modules)
```

## Responsive Behavior

- **Desktop (> 1024px)** — Full layout: sidebar + main + right panel
- **Tablet (768-1024px)** — Sidebar collapsed, right panel as overlay
- **Mobile (< 768px)** — Single column, bottom navigation, full-screen views

Uses `react-responsive` for breakpoint detection. See @DESIGN.md for responsive tokens.

## Cross-References

- @DESIGN.md — Design system tokens and components used in UI
- @API.md — GraphQL queries and Apollo Client patterns
- @SCHEMA.md — Object/field metadata that drives dynamic UI
- @GLOSSARY.md — Term definitions for all UI concepts
- @CLAUDE.md — Frontend coding conventions
