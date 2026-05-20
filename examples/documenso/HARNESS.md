---
devmd: harness
version: "1.0"
project: documenso
updated: 2026-05-13
status: minimal

ai_orchestration: false
autonomous_agents: false

ai_features:
  - name: "AI-assisted field placement"
    status: experimental
    route: "/api/ai/*"
    rate_limited: true
    description: "PDF analysis to suggest field positions"
---

# HARNESS.md — Documenso

## Status

Documenso has **no multi-agent orchestration** and no autonomous AI agents. The platform is a traditional web application. This file documents the minimal AI surface that exists and potential future directions.

## Current AI Integration

### `/api/ai/*` Route (Experimental)

A set of Hono routes mounted at `/api/ai/` on the main server. Rate-limited and behind feature flag.

```
/api/ai/
└── /suggest-fields    POST — Analyze PDF and suggest field placements
```

#### PDF Field Auto-Placement

**Purpose**: When a user uploads a PDF, the AI feature analyzes the document structure and suggests where to place signature fields, date fields, name fields, etc.

**How it works**:

1. User uploads PDF to document editor (see @SCREENS.md#document-editor)
2. Optional "Auto-place fields" button appears in field palette
3. User clicks → PDF binary sent to `/api/ai/suggest-fields`
4. AI model analyzes document layout, text content, and common patterns
5. Returns array of suggested field placements:

```typescript
interface FieldSuggestion {
  type: FieldType;        // SIGNATURE, DATE, NAME, TEXT, etc.
  page: number;
  x: number;              // percentage-based position
  y: number;
  width: number;
  height: number;
  confidence: number;     // 0-1 confidence score
  reason: string;         // "Signature line detected at bottom of page 3"
}
```

6. Suggestions rendered as ghost fields on the Konva canvas
7. User accepts, modifies, or dismisses each suggestion
8. Accepted suggestions become real fields (same as manual placement)

**Constraints**:

- Rate limit: 10 requests/minute per user
- Max PDF size: 10MB
- Does not store or learn from documents
- Falls back gracefully if AI service unavailable
- No personally identifiable information sent to external AI service (text extraction only, no images)

**Configuration**:

```bash
# Enable/disable AI features
NEXT_PUBLIC_AI_FEATURES_ENABLED=true|false

# AI provider configuration
NEXT_PRIVATE_AI_PROVIDER=openai
NEXT_PRIVATE_AI_API_KEY=sk-...
```

### What This Is NOT

- **Not an autonomous agent**: Runs only on explicit user action. No background processing.
- **Not a decision-maker**: Suggests only. User has full control to accept or reject.
- **Not always-on**: Behind feature flag. Disabled by default for self-hosted instances.
- **Not multi-agent**: Single request-response cycle. No agent coordination or planning.

## Future Considerations

If Documenso adds more AI capabilities, they could include:

### Near-Term (Feature-Level)

| Feature | Description | Architecture |
|---|---|---|
| Smart field detection | OCR + layout analysis to identify signature lines, date fields, checkboxes in scanned documents | Synchronous API route, same as current pattern |
| Document classification | Auto-detect document type (contract, NDA, invoice) and suggest matching template | Synchronous, runs at upload time |
| Recipient suggestion | Analyze document text to suggest who should sign (based on named entities) | Synchronous, opt-in per document |

### Long-Term (Agent-Level)

| Feature | Description | Would Require |
|---|---|---|
| Compliance review agent | Analyze document against regulatory requirements before sending | Background job with @RUNTIME.md pattern. Async results displayed in editor |
| Bulk processing agent | Process a folder of documents: classify, assign templates, route to recipients | Multi-step background workflow. Would need HARNESS.md upgrade to full agent spec |
| Contract negotiation assistant | Track changes between document versions, highlight material differences | Would require @AGENTS.md upgrade with proper agent definition |

### If Agents Are Added

When Documenso adds autonomous AI agents, this file should be expanded to include:

- Agent registry (names, capabilities, triggers)
- Orchestration model (sequential, parallel, event-driven)
- Safety boundaries (what agents can and cannot do)
- Human-in-the-loop gates (approval steps before agent actions)
- Resource limits (token budgets, execution timeouts)
- Audit trail integration with existing @LOGGING.md#audit-events

See @AGENTS.md for the current (minimal) agent status.

## Cross-References

- AI feature flag: @CONFIG.md#feature-flags
- Background job system (closest analogy): @RUNTIME.md
- Agent definitions: @AGENTS.md
- Field types that AI suggests: @GLOSSARY.md#field-type-reference
- Document editor where suggestions appear: @SCREENS.md#document-editor
