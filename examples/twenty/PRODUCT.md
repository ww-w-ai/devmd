---
devmd: product
version: "1.0"
name: Twenty CRM
tagline: "Open Source CRM — Alternative to Salesforce"
type: saas
license: AGPL-3.0
stage: growth
website: https://twenty.com
repo: https://github.com/twentyhq/twenty

vision: >
  Build the most flexible, open-source CRM that gives teams full control
  over their customer data and workflows — no vendor lock-in, no per-seat
  pricing traps, no black-box limitations.

target_users:
  primary:
    - label: Startup founders & ops teams
      pain: Salesforce is too expensive and complex for early-stage companies
      value: Free, self-hostable CRM with modern UX
    - label: Developers building CRM integrations
      pain: Proprietary CRM APIs are restrictive, poorly documented, and rate-limited
      value: Full source access, GraphQL API, extensible SDK
  secondary:
    - label: Enterprise teams seeking data sovereignty
      pain: Cloud CRM stores sensitive customer data on third-party servers
      value: Self-host on own infrastructure with full audit trail
    - label: Solo consultants & freelancers
      pain: Existing CRMs are overkill or too expensive for one-person shops
      value: Lightweight, fast, free tier on Twenty Cloud

value_propositions:
  - id: metadata-driven
    statement: >
      Schema defined at runtime — users create custom objects and fields
      without code changes or migrations.
  - id: open-source
    statement: >
      AGPL-3.0 licensed. Full source access. Self-host or use Twenty Cloud.
      No feature gating between editions.
  - id: developer-extensible
    statement: >
      SDK with defineObject(), defineField(), defineSkill(), defineAgent().
      Build apps on top of Twenty like Salesforce AppExchange, but open.
  - id: modern-ux
    statement: >
      Keyboard-first, command menu, inline editing, drag-and-drop kanban,
      split views — feels like a modern productivity tool, not legacy enterprise.

success_criteria:
  - metric: GitHub stars
    target: "50,000+"
    rationale: Community adoption indicator
  - metric: Self-hosted instances
    target: "10,000+"
    rationale: Actual production usage
  - metric: Monthly active workspaces (Twenty Cloud)
    target: "5,000+"
    rationale: SaaS revenue base
  - metric: SDK apps published
    target: "100+"
    rationale: Ecosystem health

monetization:
  model: open-core
  free_tier: Self-hosted, unlimited users, all core features
  paid_tiers:
    - name: Twenty Cloud
      price: Usage-based
      features: [managed hosting, automatic updates, support SLA]
    - name: Enterprise
      price: Custom
      features: [SAML SSO, advanced RBAC, dedicated support, SLA]
---

# Twenty CRM — Product Definition

## Problem

CRM software is broken. Salesforce dominates with 23% market share but costs $25-300/user/month, requires certified consultants for customization, and locks data behind proprietary APIs. HubSpot and Pipedrive offer simpler alternatives but still restrict extensibility. Teams are forced to choose between power (Salesforce) and usability (modern tools) — they cannot have both.

## Solution

Twenty is an open-source CRM that combines Salesforce-level extensibility with modern productivity-tool UX. The metadata-driven architecture lets users create custom objects, fields, and views at runtime without touching code. The SDK enables developers to build apps, AI skills, and integrations that run inside Twenty.

## Core Capabilities

### Contact & Company Management
Standard CRM objects: **Person** (contacts), **Company** (accounts), **Opportunity** (deals with pipeline stages). Rich profiles with activity timeline, notes, tasks, and attachments. See @SCHEMA.md#standard-objects.

### Custom Objects & Fields
Users define new data types at runtime. A "Custom Project" object with custom fields appears in navigation, views, filters, and API within seconds. No migration files, no deploy. See @SCHEMA.md#custom-objects.

### Views & Filters
Saved filter/sort/group combinations per object. Table view, kanban board, calendar view. Users share views with teammates or keep them private. See @UI.md#views.

### Connected Accounts
Email (IMAP) and calendar (CalDAV) sync. Messages and events appear on contact timelines automatically. See @RUNTIME.md#email-calendar-sync.

### Workflows
Visual workflow builder for automation: trigger on record change → conditions → actions (send email, create task, update field, call webhook). See @RUNTIME.md#workflow-engine.

### AI Integration
Built-in AI features powered by Vercel AI SDK. defineSkill() for custom AI capabilities, defineAgent() for autonomous agents. Supports OpenAI, Anthropic, Google, Azure, Bedrock, Mistral, x.ai. See @AGENTS.md.

### Apps & SDK
twenty-sdk enables third-party apps: defineObject(), defineField(), defineLogicFunction(). create-twenty-app scaffolder. twenty-cli for local development. See @AGENTS.md#sdk.

## Non-Goals

- **Not a marketing automation tool.** No email campaigns, landing pages, or lead scoring. Integrate with dedicated tools via API.
- **Not a project management tool.** Tasks exist for CRM context (follow-up calls, proposals). Use Linear/Jira for engineering work.
- **Not a customer support tool.** No ticket system, no knowledge base. Integrate with Zendesk/Intercom.

## Competitive Landscape

| Competitor | Strength | Twenty Advantage |
|---|---|---|
| Salesforce | Enterprise features, ecosystem | Open source, 10x cheaper, modern UX |
| HubSpot | Marketing suite integration | No feature gating, self-hostable |
| Pipedrive | Sales-focused simplicity | Custom objects, developer extensibility |
| Attio | Modern UX, flexible data model | Open source, self-hostable, SDK |
| Folk | Lightweight, contact-centric | GraphQL API, workflow engine, AI agents |

## References

- @ARCHITECTURE.md — System structure enabling extensibility
- @SCHEMA.md — Metadata-driven data model
- @API.md — Three-tier GraphQL API
- @AGENTS.md — AI and SDK integration
- @UI.md — Frontend architecture and UX patterns
