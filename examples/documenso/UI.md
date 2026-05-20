---
devmd: ui
version: "1.0"
project: documenso
updated: 2026-05-13

framework: "React Router v7"
routing: file-based
server: hono
build: vite
state_management:
  server: "tRPC + React Query"
  forms: "React Hook Form + Zod"
  global: none  # No Redux/Zustand/Jotai
i18n:
  library: lingui
  languages: 11
  strategy: "runtime catalog loading"

layouts:
  - name: authenticated
    description: "Logged-in users — sidebar nav, team context"
    routes: "/documents, /templates, /settings, /admin"
  - name: unauthenticated
    description: "Public pages — centered, minimal chrome"
    routes: "/signin, /signup, /forgot-password"
  - name: signing
    description: "Recipient signing view — full-width PDF canvas, field toolbar"
    routes: "/sign/{token}"
  - name: embedding
    description: "Embeddable signing — iframe-compatible, minimal UI"
    routes: "/embed/sign/{token}"
---

# UI.md — Documenso

## Route Structure

React Router v7 file-based routing in `apps/remix/app/routes/`.

### Authenticated Routes (Sidebar Layout)

```
/                              → Dashboard (redirect to /documents)
/documents                     → Document list (inbox, sent, drafts, completed)
/documents/{id}                → Document detail / editor
/documents/{id}/edit           → Document field editor (Konva canvas)
/documents/{id}/logs           → Audit log viewer
/templates                     → Template list
/templates/{id}                → Template detail / editor
/templates/{id}/edit           → Template field editor
/templates/{id}/direct-link    → Direct link settings
/folders                       → Folder tree view
/settings                      → User settings
/settings/profile              → Profile editor
/settings/password             → Password change
/settings/security             → 2FA, passkeys
/settings/tokens               → API token management
/settings/webhooks             → Webhook configuration
/team                          → Team settings
/team/members                  → Team member management
/team/branding                 → Team branding/logo
/admin                         → Admin panel (admin role only)
/admin/users                   → User management
/admin/documents               → All documents
/admin/stats                   → Platform statistics
```

### Unauthenticated Routes (Centered Layout)

```
/signin                        → Login form
/signup                        → Registration form
/forgot-password               → Password reset request
/reset-password/{token}        → Password reset form
/verify-email/{token}          → Email verification
/unverified-account            → Prompt to verify email
```

### Signing Routes (Full-Width Layout)

```
/sign/{token}                  → Recipient signing view
/sign/{token}/complete         → Signing complete confirmation
/d/{directLinkToken}           → Direct link signing entry
```

### Embedding Routes (Iframe Layout)

```
/embed/sign/{token}            → Embedded signing (enterprise)
/embed/complete                → Embedded completion callback
```

## Page Patterns

### Document List Page

```
type: data-table
features:
  - Tabbed filtering: Inbox | Sent | Drafts | Completed | All
  - Column sorting: Title, Status, Created, Updated
  - Bulk actions: Delete, Move to folder
  - Search: Full-text across title and recipient names
  - Pagination: Server-side, 20 per page
  - Empty state: Illustration + CTA to create first document
data_source: trpc.document.findDocuments
```

### Document Editor Page

```
type: multi-panel
layout:
  left_panel:
    width: 280px
    content: "Recipient list + Field palette"
  center_panel:
    content: "PDF canvas (Konva) with field overlays"
    interactions:
      - Drag fields from palette onto PDF pages
      - Click field to select and configure
      - Resize fields by dragging corners
      - Navigate pages via thumbnail strip
  right_panel:
    width: 320px
    content: "Field properties inspector"
    shows: "Type, assignment, validation, options"
  top_bar:
    content: "Document title, status badge, Send button"
data_source: trpc.envelope.getEnvelope
```

### Signing Flow

The signing experience for recipients:

```
Step 1: Landing
  - Recipient opens /sign/{token}
  - See document title, sender info, field count
  - "Review Document" button

Step 2: Review & Sign
  - Full-width PDF viewer (Konva canvas)
  - Fields highlighted with recipient's color
  - Sequential field navigation ("Next field" button)
  - Field types render appropriate inputs:
    - SIGNATURE → Signature pad (type/draw/upload tabs)
    - TEXT → Inline text input
    - CHECKBOX → Click to toggle
    - DROPDOWN → Select popover
    - DATE → Date picker popover
    - etc.
  - Progress indicator: "3 of 7 fields completed"

Step 3: Confirm
  - Summary of all filled fields
  - "Complete Signing" button
  - Legal disclosure text

Step 4: Done
  - Confirmation screen
  - Download signed document (if all recipients done)
  - Redirect to custom URL (if configured in DocumentMeta)
```

### Signature Pad Component

```
type: modal-dialog
tabs:
  - Type: Text input with Caveat font preview
  - Draw: Freehand canvas (Konva drawing layer)
  - Upload: Image upload with crop/resize
output: Base64 image string stored in Signature model
```

## Component Patterns

### Data Tables

All list views use a shared `DataTable` component from `@documenso/ui`:

```typescript
<DataTable
  columns={columns}          // Column definitions with sorting
  data={data}                // Server-fetched data
  pagination={pagination}    // Server-side pagination state
  onPaginationChange={...}
  sorting={sorting}
  onSortingChange={...}
  emptyState={<EmptyState />}
/>
```

### Forms

All forms use React Hook Form + Zod:

```typescript
const schema = z.object({
  title: z.string().min(1, 'Title is required'),
  recipients: z.array(recipientSchema).min(1, 'Add at least one recipient'),
});

const form = useForm<z.infer<typeof schema>>({
  resolver: zodResolver(schema),
  defaultValues: { ... },
});
```

### Toast Notifications

```typescript
import { toast } from '@documenso/ui/primitives/use-toast';

toast({ title: 'Document sent', description: '3 recipients notified' });
toast({ title: 'Error', description: error.message, variant: 'destructive' });
```

### Loading States

- **Skeleton**: Used for initial page loads (data tables, document previews)
- **Spinner**: Used for mutation actions (sending, signing)
- **Optimistic updates**: Used for field insertion during signing

## Internationalization (i18n)

- **Library**: Lingui
- **Languages**: 11 (en, de, fr, es, pt, it, nl, pl, zh, ja, ko)
- **Strategy**: Runtime catalog loading per locale
- **Pattern**:

```typescript
import { Trans } from '@lingui/macro';

<Trans>Document sent successfully</Trans>
<Trans>
  {count} of {total} fields completed
</Trans>
```

- Translation catalogs in `apps/remix/locales/`
- CI workflow validates catalog completeness on PR

## Responsive Behavior

| Breakpoint | Layout |
|---|---|
| `< 768px` | Single column, bottom sheet for panels, hamburger nav |
| `768px - 1024px` | Collapsed sidebar, two-panel editor |
| `> 1024px` | Full sidebar, three-panel editor |

Signing flow is optimized for mobile (single column, full-width fields).

## Cross-References

- Component library: @DESIGN.md#component-architecture
- Data sources (tRPC): @API.md#trpc-routers
- Route authentication: @SECURITY.md#authorization
- Field types: @GLOSSARY.md#field-type-reference
- PDF canvas technology: @DESIGN.md#pdf-canvas-konva
