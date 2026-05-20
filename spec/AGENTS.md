# AGENTS.md Specification

> Version: 0.1.0-draft | Status: Draft

## Purpose

AGENTS.md defines AI agent personas, instructions, and behavioral constraints. DevMD adopts the AAIF AGENTS.md standard (Linux Foundation). For autonomous agent control infrastructure — LLM config, RAG, tools, guardrails, memory, team coordination, and lifecycle — use `@HARNESS.md`.

## Adopted Standard

AGENTS.md is an **externally defined** standard by AAIF (Linux Foundation). DevMD extends the standard with recommended structure while maintaining backward compatibility.

## Frontmatter Schema

AGENTS.md MAY use YAML frontmatter. The AAIF standard does not mandate it, but DevMD RECOMMENDS the following for interoperability.

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | OPTIONAL | Agent system name |
| `version` | `string` | OPTIONAL | Agent definition version |

## Agent Definition

Each agent MUST be defined as a markdown section. Multiple agents MAY coexist in one file.

### Per-Agent Fields

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `role` | `string` | REQUIRED | Agent's role or title |
| `instructions` | `string` | REQUIRED | Behavioral instructions for the agent |
| `tools` | `string[]` | OPTIONAL | Tools the agent may use |
| `constraints` | `string[]` | OPTIONAL | Behavioral constraints or prohibitions |

## Recommended Sections

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## [Agent Name]` | REQUIRED | One section per agent. Contains role, instructions, tools, constraints. |

Within each agent section:

| Subsection | Presence | Content Rules |
|------------|----------|---------------|
| `### Role` | REQUIRED | Agent's purpose and responsibilities. |
| `### Instructions` | REQUIRED | Detailed behavioral instructions. |
| `### Tools` | OPTIONAL | Available tools or MCP servers. |
| `### Constraints` | OPTIONAL | What the agent MUST NOT do. |

## Cross-References

- MUST reference `@HARNESS.md` for autonomous agent control plane (LLM, RAG, tools, guardrails, memory, team, lifecycle).
- SHOULD reference `@PRODUCT.md` for product context the agent needs.
- MAY reference `@GLOSSARY.md` for domain terminology the agent must use.
- MAY reference `@BRAND.md` for voice and tone when the agent produces user-facing content.

## Validation Rules

1. Every agent section MUST have a `role` and `instructions`.
2. Agent names MUST be unique within the file.
3. `tools` entries SHOULD reference tools that exist in the agent's runtime environment.

## Conflict Detection

- Agent `constraints` MUST NOT contradict `instructions` within the same agent.
- If `@HARNESS.md` defines agents, the agent names SHOULD match between AGENTS.md and HARNESS.md.
- Agent instructions SHOULD NOT contradict rules in `@CLAUDE.md` if both files exist.

## Notes

- AGENTS.md defines **what** an agent is (persona, behavior). HARNESS.md defines **how** the agent runs (infrastructure, orchestration).
- For simple AI coding instructions without agent personas, `@CLAUDE.md` is sufficient.
- The AAIF standard is evolving. DevMD will track upstream changes.
