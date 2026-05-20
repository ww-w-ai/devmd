---
devmd: brand
version: "1.0"
project: documenso
updated: 2026-05-13

name: Documenso
pronunciation: "doc-uh-MEN-so"
tagline: "The Open Source DocuSign Alternative"
mission: "Document signing should be open, beautiful, and accessible to everyone."

brand_values:
  - openness: "Open source first. Transparency in code, process, and pricing."
  - beauty: "Signing documents should feel delightful, not bureaucratic."
  - trust: "Cryptographic integrity. SOC2 certified. Your data, your control."
  - accessibility: "Free self-hosting. Generous free tier. Multilingual (11 languages)."
  - community: "Built by and for the community. Every contributor matters."

voice:
  tone: "Friendly, confident, technically honest"
  personality: "The approachable open-source friend who happens to be great at security"
  avoid:
    - "Corporate jargon ('synergy', 'leverage', 'paradigm')"
    - "Fear-based marketing about security"
    - "Disparaging competitors by name"
    - "Overpromising features not yet built"
  prefer:
    - "Plain language over legal/technical jargon"
    - "'You' over 'users' or 'customers'"
    - "Active voice"
    - "Specific numbers over vague claims"

writing_guidelines:
  headlines: "Short, action-oriented. Lead with benefit."
  body: "Conversational but precise. One idea per paragraph."
  cta: "Clear action verb. 'Get started' > 'Learn more'."
  technical: "Accurate and current. Link to docs, not hand-wave."
  oss_messaging: "Celebrate contributors. Credit the community."
---

# BRAND.md — Documenso

## Brand Identity

Documenso is the **open-source alternative to DocuSign**. The brand stands at the intersection of:

- **Open source ethos** — Transparency, community ownership, freedom to self-host
- **Enterprise reliability** — SOC2 certified, cryptographic signing, audit trails
- **Design quality** — Modern, clean UI that makes signing feel simple

## Visual Identity

### Logo

- Primary: Documenso wordmark with document icon
- Icon-only variant for favicons and small contexts
- Available in dark and light variants

### Colors

See @DESIGN.md#colors for full token specification.

- **Primary**: Brand green — used for CTAs, success states, active elements
- **Neutral**: Slate scale — text, borders, backgrounds
- **Semantic**: Standard green/red/yellow for success/error/warning

### Typography

- **Sans-serif**: Inter (UI), Caveat (signature handwriting feel)
- See @DESIGN.md#typography for full type scale

## Voice Examples

### Good

> "Sign documents for free. Forever. No catch."

> "Your contracts never leave your server. Self-host Documenso in minutes."

> "Built by 200+ contributors who believe signing should be open."

### Bad

> "Leverage our best-in-class e-signature solution to optimize your workflow." (corporate jargon)

> "Unlike DocuSign's expensive, closed platform..." (competitor bashing)

> "Our AI-powered blockchain-secured signatures..." (buzzword overload)

## Open Source Messaging

### The Core Narrative

Document signing is **critical infrastructure**. Employment contracts, business agreements, healthcare forms — these touch everyone. Critical infrastructure should be:

1. **Auditable** — You can read every line of code
2. **Portable** — No vendor lock-in, export anytime
3. **Affordable** — Free to self-host, fair cloud pricing
4. **Trustworthy** — Cryptographic proof, not just promises

### Community Language

- "Contributors" not "volunteers"
- "Community" not "user base"
- "We built this together" — always acknowledge community
- First-time contributor PRs get a welcome message (automated via GitHub Action)

### License

**AGPL-3.0** — Ensures modifications are shared back. Enterprise features in `@documenso/ee` may have additional terms.

## Content Guidelines

### Documentation

- Start with "what" and "why" before "how"
- Include code examples for every API endpoint
- Keep docs versioned alongside code
- Assume the reader is a developer

### Social Media

- Twitter/X: Product updates, community highlights, open source advocacy
- GitHub: Technical discussions, roadmap, contributor engagement
- Discord: Community support, feature requests, casual conversation

### Error Messages

See @ERRORS.md for technical error codes. User-facing error messages should be:

- **Specific**: "Document not found" not "Something went wrong"
- **Actionable**: "Please add at least one recipient before sending"
- **Human**: "This link has expired. Request a new one." not "Error 410: GONE"

## Brand Do's and Don'ts

### Do

- Emphasize open source and self-hosting
- Show the product UI in marketing materials
- Celebrate community milestones (stars, contributors, deployments)
- Be transparent about limitations and roadmap

### Don't

- Use stock photos of people signing paper documents
- Imply features that don't exist yet
- Hide the AGPL license
- Use dark patterns in pricing or UX

## Cross-References

- Design tokens: @DESIGN.md
- Error message guidelines: @ERRORS.md
- Product positioning: @PRODUCT.md
