# DevMD — Development as Markdown

> Google standardized UI design with DESIGN.md.
> DevMD standardizes **all of software development** with .md files.

You fill in markdown. AI builds the product.

## The Problem

Every AI coding session starts the same way: explain your product, describe the schema, clarify the API patterns, define the error codes, specify the auth flow — over and over. Each session burns thousands of tokens re-establishing context that should already exist.

Meanwhile, teams scatter specs across Notion, Confluence, Figma, Slack threads, and developer brains. AI can't read any of it.

## The Solution

**25 markdown files that define your entire software project.**

One file per concern. YAML frontmatter for machines, markdown body for humans. Drop them in your repo root. Every AI tool — Claude, Cursor, Copilot, Codex — reads them instantly.

**You don't write 25 files. AI does.** `npx devmd scan` analyzes your codebase and generates all files. You review and correct. AI builds from them. More structured templates = fewer tokens per session = better output.

```
your-project/
│
│  ── Product ──
├── PRODUCT.md        # Vision, users, value proposition
├── GLOSSARY.md       # Domain terms (DDD ubiquitous language)
├── BRAND.md          # Voice, tone, copy rules
│
│  ── Structure ──
├── ARCHITECTURE.md   # System structure, layers, dependencies
├── SCHEMA.md         # Database tables, relations, indexes
├── API.md            # Endpoints, error format, versioning
├── ERRORS.md         # Error codes, exception hierarchy, retry policy
├── LOGGING.md        # Log levels, structured format, tracing
│
│  ── Interface ──
├── DESIGN.md         # Visual tokens (Google standard)
├── UI.md             # Frontend structure, layouts, components
├── SCREENS.md        # Visual reference per screen and state
├── FLOWS.md          # UX journeys + data flow chains
├── SEO.md            # SEO + GEO strategy
│
│  ── Quality ──
├── TESTING.md        # Test pyramid, coverage, scope
├── SECURITY.md       # Auth, OWASP checklist, secrets
│
│  ── Operations ──
├── INFRA.md          # Infrastructure intent (WHAT, not HOW)
├── CONFIG.md         # Environment variables, provider matrix, feature flags
├── DEVOPS.md         # CI/CD, build, deploy, backup pipelines
├── OPERATIONS.md     # Incident runbooks, SLOs, on-call, escalation
├── CHANGELOG.md      # Version history, breaking changes, migration paths
│
│  ── AI & Agents ──
├── CLAUDE.md         # Coding conventions (Anthropic standard)
├── AGENTS.md         # AI agent persona & instructions (AAIF standard)
├── HARNESS.md        # Agent control: LLM, RAG, tools, guardrails, memory
├── RUNTIME.md        # Job/process execution spec
│
│  ── Control ──
└── LIFECYCLE.md      # State machines, phase transitions, conflict detection
```

## Design Philosophy

### 1. The Checklist Effect

An empty template asks "did you define this?" for every section. Items exist, so nothing gets missed. A blank ERRORS.md with 6 empty sections is more valuable than no file at all — it tells you what you haven't decided yet.

### 2. Abstraction Saves Tokens

Write `type: card-feed` and AI already knows the grid, spacing, responsive behavior, empty state, and loading skeleton. One line replaces ~500 tokens of detailed specification. Across a project, this saves thousands of tokens per session.

### 3. Cross-References Eliminate Duplication

`@SCHEMA.md#users` in your API.md points to the single source of truth. Change it once, every reference stays current.

### 4. Overlap Is Intentional, Conflict Is a Bug

The same concept will appear in multiple files — and that's by design. A "Recipient" shows up in GLOSSARY.md (definition), SCHEMA.md (table structure), API.md (endpoint parameters), FLOWS.md (signing journey), UI.md (component), and SCREENS.md (visual). Each file describes it from a different perspective:

| File | Perspective on "Recipient" |
|------|---------------------------|
| GLOSSARY.md | What it means in the domain |
| SCHEMA.md | How it's stored |
| API.md | How it's accessed |
| SECURITY.md | What permissions it requires |
| FLOWS.md | When it appears in the journey |
| UI.md | Where it's shown on screen |
| SCREENS.md | What it looks like |

**This overlap is valuable.** Each perspective serves a different reader (PM, backend dev, frontend dev, security engineer) and a different AI task (schema migration, API generation, UI rendering).

**But contradiction is not.** If GLOSSARY.md says a Recipient can have 5 roles and SCHEMA.md defines 4, that's a conflict — one of them is wrong, and AI will produce inconsistent code.

### Conflict Detection

When authoring DevMD files, run conflict checks between overlapping sections. When a conflict is found:

1. **Identify** — Flag the two (or more) sources that disagree
2. **Surface** — Present both versions to the author with context
3. **Resolve** — Author picks the canonical version; other files update to match
4. **Reference** — The resolved fact lives in one primary file; others use `@FILE.md#section`

The primary home for each type of fact:

| Fact Type | Primary Home | Other Files Reference It |
|-----------|-------------|------------------------|
| Term definition | GLOSSARY.md | All files |
| Table structure | SCHEMA.md | API.md, ERRORS.md |
| Visual tokens | DESIGN.md | UI.md, SCREENS.md |
| Auth rules | SECURITY.md | API.md, INFRA.md |
| Error codes | ERRORS.md | API.md, LOGGING.md |
| User journey | FLOWS.md | UI.md, SCREENS.md |
| Env config | INFRA.md | RUNTIME.md, SECURITY.md |

## File Categories

### Adopted Standards (3 files)

Already established by industry leaders. DevMD adopts them as-is.

| File | Origin | Domain |
|------|--------|--------|
| CLAUDE.md | Anthropic | Coding conventions + project rules |
| AGENTS.md | AAIF (Linux Foundation) | AI agent instructions |
| DESIGN.md | Google Labs (2024) | UI design tokens |

### DevMD Original (16 files)

Filling gaps no standard has addressed.

| File | Domain |
|------|--------|
| PRODUCT.md | Product vision, target users, success criteria |
| GLOSSARY.md | Domain vocabulary (DDD ubiquitous language) |
| ARCHITECTURE.md | System structure, layers, ADRs |
| SCHEMA.md | Database tables, relations, migrations |
| API.md | Endpoints, error format, versioning |
| ERRORS.md | Error codes, retry policy, user messages |
| LOGGING.md | Log levels, structured format, tracing |
| TESTING.md | Test pyramid, coverage, scope |
| SECURITY.md | Auth, OWASP, secrets policy |
| BRAND.md | Voice, tone, copy rules |
| FLOWS.md | UX journeys + backend data chains |
| SCREENS.md | Visual reference per screen and state |
| CONFIG.md | Environment variables, provider matrix, feature flags |
| DEVOPS.md | CI/CD pipelines, build, deploy, backup |
| OPERATIONS.md | Incident runbooks, SLOs, on-call, escalation |
| CHANGELOG.md | Version history, breaking changes, migration paths |

### DevMD Firsts (6 files)

Categories where **no one has proposed a standard yet**. DevMD is the first.

| File | Domain | Why It's Empty |
|------|--------|---------------|
| **UI.md** | Frontend structure, layouts, component patterns | DESIGN.md covers visuals only; structure had no spec |
| **SEO.md** | SEO + GEO (AI search optimization) | SEO was tools and practices, never a declarative spec |
| **INFRA.md** | Infrastructure intent definition | Terraform is HOW (code); the WHAT (intent) had no spec |
| **HARNESS.md** | Agent control infrastructure | Agent definitions (AGENTS.md) exist; control plane spec didn't |
| **LIFECYCLE.md** | State-driven control and transitions | Static specs exist; dynamic state management didn't |
| **RUNTIME.md** | Agent/job execution specification | Agent definitions exist (BMAD); execution specs didn't |

## How Files Relate

```
PRODUCT.md ── GLOSSARY.md ── BRAND.md ──────────── Why we build + what we call things

ARCHITECTURE.md ── SCHEMA.md ── API.md ─────────── How it's structured
    │                 │           │
ERRORS.md ── LOGGING.md ── TESTING.md ──────────── How it behaves
    │
SECURITY.md ── INFRA.md ── CONFIG.md ───────────── How it's secured + configured
    │
DEVOPS.md ── OPERATIONS.md ── CHANGELOG.md ─────── How it's operated

DESIGN.md ── UI.md ── SCREENS.md ───────────────── How it looks (tokens → structure → result)
    │           │          │
FLOWS.md ── SEO.md ─────────────────────────────── How it moves + speaks

CLAUDE.md ── AGENTS.md ── HARNESS.md ───────────── How AI works with it
    │                        │
RUNTIME.md ── LIFECYCLE.md ──────────────────────── How it runs + transitions
```

## Why 25 Files?

We documented two real open-source projects — [Twenty CRM](examples/twenty/) (44K stars, full-stack) and [Documenso](examples/documenso/) (9K stars, document signing) — using 17 files and performed a rigorous gap analysis.

**106 gaps found.** Adding 8 files resolved 93% of them.

| Gap Type | Count | Root Cause | Solved By |
|----------|-------|-----------|-----------|
| End-to-end flows fragmented | 25 | No file owned the full journey | FLOWS.md |
| State transitions undocumented | 12 | Static specs, no dynamic control | LIFECYCLE.md |
| Env vars / config matrix missing | 10 | No structured config spec | CONFIG.md |
| Deployment / operations gaps | 10 | No ops perspective | DEVOPS.md + OPERATIONS.md |
| Agent control plane missing | 8 | AGENTS.md = persona only | HARNESS.md |
| Visual reference missing | 5 | Tokens ≠ rendered result | SCREENS.md |
| Evolution history lost | 3 | Current-state snapshots only | CHANGELOG.md |
| Depth insufficient in existing | 34 | Files need more detail | Existing file expansion |

The remaining 7% are runtime-specific implementation details that belong in code, not specs. [Full analysis →](docs/gap-analysis.md)

## Who Writes These Files?

**Not you. AI does.**

```
npx devmd scan     → AI analyzes your codebase, generates 25 files
You                → Review, correct, fill in business context
AI                 → Reads DevMD files, builds features with full context
```

This is exactly what we did with Twenty and Documenso: AI scanned the projects and generated 17 files (8,982 lines) automatically. The 25-file set gives AI more structure, which means fewer tokens per session and more accurate output.

## Project Tiers

You don't need all 25 files. Pick a tier:

| Tier | Files | Use When |
|------|-------|----------|
| **Starter** (8) | PRODUCT, DESIGN, UI, SEO, BRAND, CLAUDE, SCREENS, INFRA | Static sites, landing pages |
| **App** (16) | + SCHEMA, API, ERRORS, SECURITY, TESTING, LOGGING, FLOWS, ARCHITECTURE | SaaS, web apps |
| **AI-Native** (21) | + AGENTS, HARNESS, LIFECYCLE, RUNTIME, CONFIG | AI-powered products |
| **Enterprise** (25) | + DEVOPS, OPERATIONS, CHANGELOG, remaining | Full-stack teams at scale |

## Getting Started

### Install the plugin

```bash
claude plugin install devmd@ww-w-ai
```

### New project — write specs from scratch

```
/devmd-guide              # Interactive walkthrough — pick tier, answer questions, generate files
/devmd-guide app          # Start with App tier (16 files)
/devmd-guide SCHEMA.md    # Write a single file
```

The guide walks you through 7 waves in dependency order, asking 3–5 questions per file. You answer, AI writes the spec.

### Existing project — auto-generate from code

```
/devmd-scan               # Analyze codebase, generate all files for detected tier
/devmd-scan SCHEMA.md     # Generate a single file
```

### Verify specs against source

```
/devmd-gap-analysis                   # Full coverage + accuracy + consistency check
/devmd-gap-analysis SCHEMA.md         # Single file
/devmd-gap-analysis --consistency-only # Cross-file conflict detection only
```

## Examples

Real-world projects documented with DevMD:

| Project | Stars | Type | Files | Lines |
|---------|-------|------|-------|-------|
| [Twenty CRM](examples/twenty/) | ~44K | Full-stack CRM | 25 | ~6,000 |
| [Documenso](examples/documenso/) | ~9K | Document signing | 25 | ~5,500 |

## Competitive Landscape

```
Google DESIGN.md    = One file for design tokens
BMAD-METHOD         = Agent definition best practices
Kiro (AWS)          = IDE-locked 3-file system ($20-200/mo)
specs.md (fabriqai) = Methodology framework

DevMD               = Open standard covering all of software development
                      Free. Tool-agnostic. 25 files.
```

## Token Savings

A typical AI coding session spends ~4,000 tokens per page re-establishing context. With DevMD:

- `type: card-feed` → AI knows grid + spacing + empty state + loading (~500 tokens saved)
- `@SCHEMA.md#users` → No need to re-describe the table (~200 tokens saved)
- 25 files loaded once → Every subsequent task starts with full context

For a 10-page app with 20 AI sessions: **~80,000 tokens saved**. That's real money and real time.

## Contributing

DevMD is an open standard. We want every framework, every AI tool, and every team to adopt it — but only if it actually works.

The best way to contribute: **pick a real project and document it with DevMD.** Then tell us what was missing, what was redundant, and what you'd change.

## License

MIT

---

**DevMD** is a [ww-w.ai](https://ww-w.ai) project.
Mission: "Do it once, share with all."
