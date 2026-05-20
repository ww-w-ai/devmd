---
devmd: product
version: "1.0"
project: documenso
updated: 2026-05-13

name: Documenso
tagline: "The Open Source DocuSign Alternative"
license: AGPL-3.0
website: https://documenso.com
github: https://github.com/documenso/documenso
status: production

target_users:
  primary:
    - segment: small-medium-businesses
      pain: "Paying $25-50/user/month for DocuSign/Adobe Sign"
      job: "Get documents signed quickly with legal validity"
    - segment: developers
      pain: "No API-first, self-hostable signing solution"
      job: "Embed document signing into their own apps"
    - segment: enterprises-privacy-conscious
      pain: "Sensitive documents on third-party servers"
      job: "Self-host signing infrastructure with full data control"
  secondary:
    - segment: open-source-advocates
      pain: "No credible OSS alternative to proprietary signing"
      job: "Support and use open-source infrastructure"

value_proposition:
  one_liner: "Document signing should be beautiful, open, and accessible to everyone."
  differentiators:
    - "Fully open source (AGPL-3.0) — audit every line"
    - "Self-hostable — your data never leaves your servers"
    - "Cryptographic PDF signing — legally binding with HSM support"
    - "API-first — embed signing into any application via tRPC + OpenAPI"
    - "One-click deploy — Railway, Render, Koyeb, Elestio"
    - "SOC2 certified managed hosting available"

success_criteria:
  north_star: "Monthly signed documents"
  metrics:
    - name: monthly_signed_documents
      target: "growth > 20% MoM"
    - name: self_host_installs
      target: "tracked via telemetry opt-in"
    - name: api_integrations
      target: "active API tokens with > 10 calls/month"
    - name: github_stars
      current: "~10K"
    - name: community_contributors
      target: "growing monthly"

pricing:
  model: open-core
  tiers:
    - name: Self-Hosted (Free)
      price: "$0"
      includes: "Full platform, unlimited documents, community support"
    - name: Cloud
      price: "Free tier + paid plans"
      includes: "Managed hosting, automatic updates, email support"
    - name: Enterprise
      price: "Custom"
      includes: "SSO/SAML, advanced audit, SLA, dedicated support, embeddable signing"

competitors:
  - name: DocuSign
    weakness: "Expensive, closed source, vendor lock-in"
  - name: Adobe Sign
    weakness: "Complex, expensive, Adobe ecosystem dependent"
  - name: PandaDoc
    weakness: "Closed source, limited self-hosting"
  - name: SignNow
    weakness: "Closed source, limited API"
---

# PRODUCT.md — Documenso

## Vision

Document signing is critical infrastructure. It should not be locked behind proprietary vendors. Documenso exists to make document signing open, beautiful, and accessible to everyone — from solo freelancers to enterprises handling millions of signatures.

## Problem Statement

1. **Cost**: DocuSign charges $25-50/user/month. Teams of 20+ pay $6,000-12,000/year for a feature that should be commodity infrastructure.
2. **Data sovereignty**: Sensitive contracts (employment, M&A, healthcare) are uploaded to third-party servers with no self-hosting option.
3. **Developer experience**: Existing signing APIs are clunky REST APIs with inconsistent docs. No modern tRPC/OpenAPI-first approach exists.
4. **Lock-in**: Proprietary formats and workflows make migration nearly impossible.

## Solution

A fully open-source document signing platform that is:

- **Self-hostable** with Docker in minutes
- **API-first** with tRPC + OpenAPI for embedding into any application
- **Cryptographically secure** with PDF digital signatures (local P12 or Google Cloud HSM)
- **Beautiful** with a modern React UI built on Radix/shadcn components
- **Compliant** with SOC2 certification for managed hosting

## Core Workflow

See @GLOSSARY.md#envelope-lifecycle for domain terms.

```
Create Envelope (DRAFT)
  → Add Document Items
  → Add Recipients (SIGNER, APPROVER, VIEWER, CC, ASSISTANT)
  → Add Fields (SIGNATURE, TEXT, DATE, CHECKBOX, etc.)
  → Send (PENDING)
  → Recipients Sign in order (PARALLEL or SEQUENTIAL)
  → Seal Document (cryptographic PDF signing)
  → Complete (COMPLETED)
```

Alternative flows: Rejection, Expiration, Cancellation.

## Key Features

| Feature | Description |
|---|---|
| Document signing | Upload PDF, add fields, send for signature |
| Templates | Reusable document templates with pre-configured fields |
| Direct links | Public signing links without email invitation |
| Embedding | Embed signing UI in third-party applications |
| Teams & orgs | Multi-team organization with RBAC |
| Audit trail | 18 event types with timestamps and IP addresses |
| API | Full tRPC + OpenAPI for programmatic access |
| Webhooks | Event notifications to external systems |
| i18n | 11 languages via Lingui |
| Background jobs | Async processing via swappable providers |

## Roadmap Signals

- Embeddable signing (enterprise feature)
- Advanced template workflows
- Expanded OAuth/OIDC provider support
- Mobile-optimized signing experience
