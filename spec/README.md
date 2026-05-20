# DevMD Specification — Index

> Version: 0.1.0-draft  
> Status: Draft  
> License: CC BY 4.0  

## Overview

DevMD standardizes software development knowledge as Markdown files with YAML frontmatter. This directory contains the **normative specification** for every DevMD file. Keywords MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY follow [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

## Common Conventions

1. **Encoding.** Every DevMD file MUST be UTF-8 encoded, LF line endings.
2. **Frontmatter.** YAML frontmatter MUST appear between `---` fences at the top of the file.
3. **Body.** Markdown body follows immediately after the closing `---`.
4. **Placement.** Files MUST be placed in the project root unless overridden by `devmd.config.json`.
5. **Cross-references.** Use `@FILE.md#section` to reference another DevMD file and section.
6. **Token interpolation.** Within YAML, `{file.field}` resolves a value from another DevMD file's frontmatter.
7. **RFC 2119.** Normative keywords are UPPER CASE throughout these specs.

## Specification Files

### Adopted Standards (4)

| File | Spec | Origin |
|------|------|--------|
| CLAUDE.md | [spec/CLAUDE.md](CLAUDE.md) | Anthropic (DevMD adoption doc) |
| AGENTS.md | [spec/AGENTS.md](AGENTS.md) | AAIF / Linux Foundation (DevMD adoption doc) |
| DESIGN.md | [spec/DESIGN.md](DESIGN.md) | Google Labs (DevMD adoption doc) |
| SECURITY.md | [spec/SECURITY.md](SECURITY.md) | GitHub (DevMD extension) |

### DevMD Originals (9)

| File | Spec | Domain |
|------|------|--------|
| PRODUCT.md | [spec/PRODUCT.md](PRODUCT.md) | Product vision, users, value |
| GLOSSARY.md | [spec/GLOSSARY.md](GLOSSARY.md) | Domain terminology (DDD) |
| BRAND.md | [spec/BRAND.md](BRAND.md) | Voice, tone, copy rules |
| ARCHITECTURE.md | [spec/ARCHITECTURE.md](ARCHITECTURE.md) | System structure, layers, ADRs |
| SCHEMA.md | [spec/SCHEMA.md](SCHEMA.md) | Data models, relations, migrations |
| API.md | [spec/API.md](API.md) | Endpoints, auth, errors |
| ERRORS.md | [spec/ERRORS.md](ERRORS.md) | Error codes, retry, hierarchy |
| LOGGING.md | [spec/LOGGING.md](LOGGING.md) | Log format, levels, tracing |
| TESTING.md | [spec/TESTING.md](TESTING.md) | Test pyramid, coverage, scope |

### DevMD Unique Proposals (10)

| File | Spec | Domain |
|------|------|--------|
| UI.md | [spec/UI.md](UI.md) | Frontend structure, layout, flow |
| SCREENS.md | [spec/SCREENS.md](SCREENS.md) | Visual reference per screen/state |
| FLOWS.md | [spec/FLOWS.md](FLOWS.md) | UX journeys + data flows |
| SEO.md | [spec/SEO.md](SEO.md) | SEO + GEO strategy |
| INFRA.md | [spec/INFRA.md](INFRA.md) | Infrastructure intent |
| CONFIG.md | [spec/CONFIG.md](CONFIG.md) | Env vars, providers, feature flags |
| DEVOPS.md | [spec/DEVOPS.md](DEVOPS.md) | CI/CD, build, deploy, backup |
| OPERATIONS.md | [spec/OPERATIONS.md](OPERATIONS.md) | Incidents, SLOs, runbooks |
| CHANGELOG.md | [spec/CHANGELOG.md](CHANGELOG.md) | Version history, migrations |
| RUNTIME.md | [spec/RUNTIME.md](RUNTIME.md) | Job execution, queues, cron |

### DevMD First (2)

| File | Spec | Domain |
|------|------|--------|
| HARNESS.md | [spec/HARNESS.md](HARNESS.md) | Agent control plane (LLM, RAG, tools, guardrails) |
| LIFECYCLE.md | [spec/LIFECYCLE.md](LIFECYCLE.md) | State-driven control, phase transitions |

## Type System

Specs use the following type annotations:

| Type | Description |
|------|-------------|
| `string` | UTF-8 text |
| `string[]` | Array of strings |
| `number` | Integer or float |
| `boolean` | `true` or `false` |
| `url` | Valid URL string |
| `markdown` | Prose block (Markdown-formatted) |
| `enum(a\|b\|c)` | One of the listed values |
| `map<string, T>` | Key-value map with string keys and values of type T |
| `@reference` | Cross-file reference using `@FILE.md#section` syntax |
| `any` | Unconstrained type |

## Versioning

Spec versions follow SemVer. Breaking changes to REQUIRED fields increment MAJOR.
