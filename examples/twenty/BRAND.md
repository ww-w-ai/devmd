---
devmd: brand
version: "1.0"
project: Twenty CRM

identity:
  name: Twenty
  tagline: "Open Source CRM"
  mission: "Build the CRM you actually want to use."
  values:
    - openness: "Open source, open community, open roadmap"
    - craftsmanship: "Every pixel and every API matters"
    - simplicity: "Powerful features, simple experience"
    - respect: "Respect user data, respect user time, respect contributors"

voice:
  tone: "Confident but approachable. Technical but not jargon-heavy."
  personality:
    - Direct — say what you mean, no corporate fluff
    - Helpful — guide, don't lecture
    - Technical — developers are the audience, don't dumb down
    - Inclusive — open source means everyone is welcome
  avoid:
    - Corporate buzzwords ("synergy", "leverage", "paradigm shift")
    - Hype language ("revolutionary", "game-changing", "disruptive")
    - Aggressive sales language ("buy now", "limited time", "don't miss out")
    - Condescending explanations — assume readers are smart

writing_rules:
  sentence_style: Active voice preferred. Short sentences for clarity.
  formatting: "Markdown everywhere. Headers for structure. Lists for steps."
  technical_docs: "Code examples first, explanations second."
  error_messages: "Say what went wrong + what to do about it."
  commit_messages: "Conventional Commits (feat:, fix:, chore:, docs:, refactor:)"
  pr_descriptions: "What changed, why, how to test."

terminology:
  preferred:
    - "open source" (not "open-source" as adjective before noun)
    - "self-host" (verb), "self-hosted" (adjective)
    - "workspace" (not "tenant" or "organization" in user-facing text)
    - "custom object" (not "custom entity" or "custom table")
  see: "@GLOSSARY.md for full term list"

community:
  channels:
    - GitHub Discussions — feature requests, questions
    - Discord — real-time chat, support
    - X/Twitter — announcements, community highlights
  contributor_tone: >
    Welcome all contributions. "Good first issue" labels for newcomers.
    Constructive code review — explain the "why" behind feedback.
    Celebrate contributions publicly.
  open_source_messaging: >
    Twenty is AGPL-3.0. Free to use, free to self-host, free to modify.
    Contributions welcome. We build in the open — roadmap, issues, and
    discussions are all public.
---

# Twenty CRM Brand

## Voice Examples

### Good

> "Twenty gives you a CRM that works the way you think. Create custom objects, build views that match your workflow, and own your data — no vendor lock-in."

> "Self-host Twenty in minutes with Docker Compose. Four containers: server, worker, PostgreSQL, Redis. That's it."

> "Something went wrong creating that person record. Check that the email format is valid and try again. If this keeps happening, open an issue on GitHub."

### Bad

> "Twenty is a revolutionary, game-changing CRM platform that leverages cutting-edge AI to disrupt the traditional CRM paradigm." (Too many buzzwords)

> "Simply deploy our solution to realize synergies across your GTM motion." (Corporate jargon)

> "You need to configure the OIDC provider metadata endpoint URL in the authentication module settings page." (Too technical for a UI message — simplify)

## Messaging Hierarchy

1. **Open source CRM** — Always lead with this. It is the primary differentiator.
2. **Extensible** — Custom objects, SDK, AI agents. You can build on top of Twenty.
3. **Modern UX** — Feels like a productivity tool, not legacy enterprise software.
4. **Developer-first** — GraphQL API, TypeScript SDK, self-hostable.
5. **Data ownership** — Your data, your server, your rules.

## Cross-References

- @GLOSSARY.md — Canonical terminology for all domains
- @PRODUCT.md — Product positioning and competitive landscape
- @DESIGN.md — Visual brand expression (colors, typography)
