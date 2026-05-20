---
devmd: agents
version: "1.0"
project: documenso
updated: 2026-05-13
status: minimal

ai_agents: false
ai_features:
  - name: "AI-assisted field placement"
    status: experimental
    description: "Suggests field positions on uploaded PDFs using document analysis"
    route: "/ai"
    integration: "Mounted on Hono server at /ai route"
---

# AGENTS.md — Documenso

## Status

Documenso does **not** currently employ AI agents as part of its core architecture. The platform is a traditional web application with server-side business logic.

## Existing AI Integration

There is a minimal AI feature mounted at the `/ai` Hono route:

- **AI-assisted field placement**: Analyzes uploaded PDF documents to suggest optimal positions for signature fields, text fields, and other form elements.
- This is an **optional, experimental feature** — not an autonomous agent.
- It runs synchronously within request context, not as a background agent.

## Future Considerations

If Documenso adds AI agents in the future, they would likely cover:

1. **Document classification**: Auto-detect document type (contract, NDA, invoice) and suggest template
2. **Smart field detection**: OCR + NLP to identify where signatures and fields should be placed
3. **Recipient suggestion**: Based on document content, suggest who should sign
4. **Compliance checking**: Verify document meets regulatory requirements before sending

These would be defined with proper agent specifications. See @RUNTIME.md for how background jobs (the closest analogy to agents) currently operate.

## Cross-References

- Background job system (agent-like execution): @RUNTIME.md#background-jobs
- API route where AI is mounted: @ARCHITECTURE.md#server-architecture
- Product roadmap signals: @PRODUCT.md#roadmap-signals
