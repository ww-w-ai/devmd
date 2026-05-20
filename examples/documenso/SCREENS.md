---
devmd: screens
version: "1.0"
project: documenso
updated: 2026-05-13

screens:
  - name: dashboard
    route: /documents
    layout: sidebar
    states: [default, empty, loading, dark]

  - name: document-editor
    route: /documents/{id}/edit
    layout: three-panel
    states: [default, field-selected, sending, dark]

  - name: signing-view
    route: /sign/{token}
    layout: full-width
    states: [reviewing, filling, signature-pad, completed]

  - name: settings
    route: /settings
    layout: sidebar
    states: [default, dark]

  - name: onboarding
    route: /signup → /onboarding
    layout: centered
    states: [step-1-profile, step-2-team, step-3-first-document]
---

# SCREENS.md — Documenso

## Dashboard (Document List)

**Route**: `/documents`
**Layout**: Sidebar + Content area
**Data Source**: `trpc.document.findDocuments`

### Default State

```
┌──────────────────────────────────────────────────────────────────┐
│  ☰ Documenso                              🔔  👤 John Doe  ▾   │
├──────────┬───────────────────────────────────────────────────────┤
│          │  Documents                        [+ New Document]   │
│ 📄 Docs  │                                                      │
│ 📋 Templ │  [ Inbox | Sent | Drafts | Completed | All ]         │
│ 📁 Foldr │                                                      │
│          │  ┌─────────────────────────────────────────────────┐  │
│ ──────── │  │ ● Service Agreement        PENDING   May 13    │  │
│ ⚙ Settngs│  │   3 recipients · 2/3 signed                    │  │
│ 👥 Team  │  ├─────────────────────────────────────────────────┤  │
│          │  │ ● NDA Template Copy        DRAFT     May 12    │  │
│          │  │   1 recipient · Not sent                        │  │
│          │  ├─────────────────────────────────────────────────┤  │
│          │  │ ● Employment Contract      COMPLETED May 11    │  │
│          │  │   2 recipients · All signed                     │  │
│          │  └─────────────────────────────────────────────────┘  │
│          │                                                      │
│          │  Page 1 of 5    [< Prev]  1  2  3  4  5  [Next >]   │
└──────────┴───────────────────────────────────────────────────────┘
```

**Status Badges** (color-coded per @DESIGN.md#theme-system):
- `DRAFT` — muted/gray
- `PENDING` — primary/green (animated pulse for active signing)
- `COMPLETED` — primary/green (solid)
- `REJECTED` — destructive/red

### Empty State

```
┌───────────────────────────────────────┐
│                                       │
│         [Document illustration]       │
│                                       │
│     No documents yet                  │
│     Create your first document        │
│     to get started.                   │
│                                       │
│        [+ New Document]               │
│                                       │
└───────────────────────────────────────┘
```

### Loading State

- Sidebar renders immediately (static navigation)
- Content area shows 5x skeleton rows (shimmer animation)
- Tab bar shows but is non-interactive until data loads

### Dark Mode

- Background: `hsl(var(--background))` switches to dark slate
- Status badges maintain same hue but adjusted for dark contrast
- Sidebar separator uses `hsl(var(--border))` dark variant
- See @DESIGN.md#css-custom-properties for dark theme values

---

## Document Editor

**Route**: `/documents/{id}/edit`
**Layout**: Three-panel (sidebar + PDF canvas + inspector)
**Data Source**: `trpc.envelope.getEnvelope`

### Default State (No Field Selected)

```
┌──────────────────────────────────────────────────────────────────┐
│  ← Back   Service Agreement          DRAFT    [Save] [Send ▸]  │
├──────────┬──────────────────────────────────┬───────────────────┤
│Recipients│                                  │ Field Properties  │
│          │    ┌────────────────────────┐     │                   │
│ 🟢 Alice │    │                        │     │ (No field         │
│   Signer │    │      PDF Page 1        │     │  selected)        │
│ 🔵 Bob   │    │                        │     │                   │
│   Signer │    │   [Sig]  [Date]        │     │ Click a field     │
│ 🟡 Carol │    │                        │     │ on the document   │
│   Viewer │    │        [Name]          │     │ to configure it.  │
│          │    │                        │     │                   │
│──────────│    └────────────────────────┘     │                   │
│ Field    │                                  │                   │
│ Palette  │    [1] [2] [3]  page nav         │                   │
│          │                                  │                   │
│ Signature│                                  │                   │
│ Initials │                                  │                   │
│ Text     │                                  │                   │
│ Date     │                                  │                   │
│ Email    │                                  │                   │
│ Checkbox │                                  │                   │
│ Dropdown │                                  │                   │
│ Number   │                                  │                   │
│ Radio    │                                  │                   │
└──────────┴──────────────────────────────────┴───────────────────┘
```

### Field Selected State

Right panel populates with field configuration:

```
┌───────────────────┐
│ Field Properties   │
│                    │
│ Type: SIGNATURE    │
│ Assigned: 🟢 Alice │
│                    │
│ Required: [✓]      │
│ Page: 1            │
│ Position: 340, 520 │
│ Size: 200 x 60     │
│                    │
│ [Delete Field]     │
└───────────────────┘
```

### Sending State

- "Send" button shows spinner + "Sending..."
- Overlay dimmer on PDF canvas
- Recipient list shows checkmarks as emails queue
- On success: redirect to document detail with "Document sent" toast
- On failure: error toast with specific message from @ERRORS.md

### Responsive Behavior

| Breakpoint | Layout |
|---|---|
| `< 768px` | Single column. PDF viewer full-width. Recipient list and field palette in bottom sheet. Inspector in dialog |
| `768px - 1024px` | Two columns. Left panel + PDF canvas. Inspector in dialog on field click |
| `> 1024px` | Full three-panel layout as shown above |

See @UI.md#responsive-behavior for breakpoint definitions.

---

## Signing View (Recipient Perspective)

**Route**: `/sign/{token}`
**Layout**: Full-width, no sidebar (minimal chrome for focus)
**Data Source**: `trpc.recipient.getRecipient` (by token)

### State 1: Reviewing

```
┌──────────────────────────────────────────────────────────────────┐
│  Documenso                                     Sent by: Alice   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Service Agreement                                               │
│  7 fields to complete                                            │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                                                            │  │
│  │                    PDF Page 1 Preview                      │  │
│  │              (fields dimmed, not interactive)              │  │
│  │                                                            │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│                    [Review Document]                              │
│                                                                  │
│  By continuing, you agree to use electronic signatures.          │
└──────────────────────────────────────────────────────────────────┘
```

### State 2: Filling Fields

```
┌──────────────────────────────────────────────────────────────────┐
│  Service Agreement                    3 of 7 fields completed   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                                                            │  │
│  │                    PDF Page 1                              │  │
│  │                                                            │  │
│  │   Name: [Bob Smith_________]  ← active field (highlighted) │  │
│  │                                                            │  │
│  │   Date: [2026-05-13]                                       │  │
│  │                                                            │  │
│  │   Signature: [ Click to sign ]  ← pending (pulsing border) │  │
│  │                                                            │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  [← Previous Field]                        [Next Field →]        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

- Fields assigned to this recipient are highlighted in their assigned color
- Fields assigned to other recipients are visible but grayed out
- "Next Field" button scrolls to and focuses the next unfilled field
- Progress bar fills as fields are completed

### State 3: Signature Pad

```
┌──────────────────────────────────────────────────────────────────┐
│  Sign Document                                           [✕]    │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [ Type ]  [ Draw ]  [ Upload ]                                  │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                                                            │  │
│  │              (Freehand drawing canvas)                     │  │
│  │                                                            │  │
│  │                   ~ Bob Smith ~                             │  │
│  │                                                            │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  [Clear]                                    [Insert Signature]   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

- **Type tab**: Text input with Caveat font preview (see @DESIGN.md#typography)
- **Draw tab**: Konva freehand canvas (see @DESIGN.md#pdf-canvas-konva)
- **Upload tab**: Image upload with crop/resize
- Output: Base64 image stored in `Signature` model (see @SCHEMA.md)

### State 4: Completed

```
┌──────────────────────────────────────────────────────────────────┐
│  Documenso                                                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│                      ✓ Signing Complete                          │
│                                                                  │
│  You have successfully signed "Service Agreement".               │
│                                                                  │
│  All recipients have signed. Your document is ready.             │
│                                                                  │
│              [Download Signed Document]                           │
│                                                                  │
│  A copy has been sent to your email.                             │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

- Download button appears only when all recipients have completed and PDF is sealed
- If other recipients are still pending: "You're done. We'll email you the signed document once everyone has signed."
- Custom redirect URL (if configured in `DocumentMeta.redirectUrl`)

---

## Settings

**Route**: `/settings/*`
**Layout**: Sidebar + Content with settings sub-navigation

### Settings Navigation

```
┌──────────┬───────────────────────────────────────────────────────┐
│          │  Settings                                             │
│ (main    │                                                      │
│  sidebar)│  [ Profile | Password | Security | Tokens | Webhooks ]│
│          │                                                      │
│          │  ┌─────────────────────────────────────────────────┐  │
│          │  │                                                 │  │
│          │  │  (Active settings panel content)                │  │
│          │  │                                                 │  │
│          │  └─────────────────────────────────────────────────┘  │
└──────────┴───────────────────────────────────────────────────────┘
```

**Sub-pages**:

| Tab | Route | Content |
|---|---|---|
| Profile | `/settings/profile` | Name, email, avatar, public profile URL. Form with React Hook Form + Zod validation |
| Password | `/settings/password` | Current password + new password. Strength indicator |
| Security | `/settings/security` | 2FA toggle (TOTP setup with QR code), passkey management (register/delete), active sessions list |
| Tokens | `/settings/tokens` | API token list (name, created, last used). Generate new with scoped permissions. Revoke with confirmation dialog. Token prefix: `dt_` |
| Webhooks | `/settings/webhooks` | Webhook URL list with event subscriptions. Test button sends ping. Status indicator (green/yellow/red based on recent delivery success) |

**Team Settings** (`/team/*`):

| Tab | Route | Content |
|---|---|---|
| General | `/team` | Team name, URL, avatar |
| Members | `/team/members` | Member list with role badges (ADMIN/MANAGER/MEMBER). Invite by email. Role change dropdown |
| Branding | `/team/branding` | Custom logo upload. Applied to signing emails and PDF seal |
| Billing | (linked) | Redirect to Stripe billing portal. Plan badge shown in sidebar |

---

## Onboarding Wizard

**Route**: Post-signup redirect
**Layout**: Centered, card-based, stepper indicator

### Step 1: Profile Setup

```
┌──────────────────────────────────────────┐
│           Welcome to Documenso           │
│                                          │
│  Step 1 of 3  ●──○──○                    │
│                                          │
│  What's your name?                       │
│  [________________________]              │
│                                          │
│  Upload a signature                      │
│  (Optional — you can do this later)      │
│  [ Type | Draw | Upload ]                │
│  ┌──────────────────────┐                │
│  │                      │                │
│  └──────────────────────┘                │
│                                          │
│                           [Continue →]   │
└──────────────────────────────────────────┘
```

### Step 2: Create or Join Team

```
┌──────────────────────────────────────────┐
│           Set Up Your Workspace          │
│                                          │
│  Step 2 of 3  ●──●──○                    │
│                                          │
│  Team name                               │
│  [________________________]              │
│                                          │
│  Team URL                                │
│  documenso.com/t/[____________]          │
│                                          │
│  (Or skip to use personal workspace)     │
│                                          │
│  [← Back]                 [Continue →]   │
└──────────────────────────────────────────┘
```

### Step 3: First Document

```
┌──────────────────────────────────────────┐
│           Send Your First Document       │
│                                          │
│  Step 3 of 3  ●──●──●                    │
│                                          │
│  Upload a document to get started:       │
│                                          │
│  ┌──────────────────────────────┐        │
│  │                              │        │
│  │    Drag & drop a PDF here    │        │
│  │    or click to browse        │        │
│  │                              │        │
│  └──────────────────────────────┘        │
│                                          │
│  (Or explore templates)                  │
│  [Browse Templates →]                    │
│                                          │
│  [← Back]          [Skip & Go to Dashboard]│
└──────────────────────────────────────────┘
```

---

## Screen State Matrix

| Screen | Default | Empty | Loading | Error | Dark Mode |
|---|---|---|---|---|---|
| Dashboard | Document list with tabs | Illustration + CTA | 5x skeleton rows | Error banner + retry | Full dark theme |
| Document Editor | 3-panel with PDF | "Upload a document" prompt | PDF skeleton + sidebar skeleton | Toast notification | Dark canvas bg |
| Signing View | PDF with highlighted fields | N/A (always has document) | Full-page spinner | Error page with sender contact | Dark supported |
| Settings | Form fields populated | N/A (forms have defaults) | Form skeletons | Inline field errors | Dark supported |
| Onboarding | Step 1 active | N/A (wizard always starts) | N/A (local state only) | Inline validation | Dark supported |

## Cross-References

- Design tokens and theme: @DESIGN.md#theme-system
- Route structure: @UI.md#route-structure
- Flow journeys mapped to screens: @FLOWS.md
- Component patterns (DataTable, forms): @UI.md#component-patterns
- Responsive breakpoints: @UI.md#responsive-behavior
- Field type rendering: @GLOSSARY.md#field-type-reference
