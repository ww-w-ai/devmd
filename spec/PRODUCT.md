# PRODUCT.md Specification

> Version: 0.1.0-draft | Status: Draft

## Purpose

PRODUCT.md defines the product vision, target users, value propositions, and success criteria.

## Frontmatter Schema

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Product name |
| `tagline` | `string` | REQUIRED | One-line description. MUST NOT exceed 120 characters. |
| `problem` | `string` | REQUIRED | Core problem being solved |
| `solution` | `string` | REQUIRED | How the product solves the problem |
| `target_users` | `TargetUser[]` | REQUIRED | Min 1 entry. See TargetUser below. |
| `value_propositions` | `string[]` | REQUIRED | Min 1 entry. Key value props. |
| `success_metrics` | `SuccessMetric[]` | OPTIONAL | Measurable outcomes |
| `competitors` | `Competitor[]` | OPTIONAL | Known competitors |
| `pricing` | `map<string, PricingTier>` | OPTIONAL | Pricing tiers keyed by tier name |

### TargetUser

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Persona name or segment label |
| `description` | `string` | REQUIRED | Who this user is |
| `pain_points` | `string[]` | REQUIRED | Problems they face. Min 1. |

### SuccessMetric

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `metric` | `string` | REQUIRED | What is measured |
| `target` | `string` | REQUIRED | Target value |
| `timeframe` | `string` | OPTIONAL | When the target should be met |

### Competitor

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Competitor name |
| `url` | `url` | OPTIONAL | Competitor URL |
| `positioning` | `string` | REQUIRED | How they position vs. us |

### PricingTier

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `price` | `string` | REQUIRED | Price (e.g., "$9/mo", "free") |
| `features` | `string[]` | REQUIRED | Features in this tier. Min 1. |

## Body Sections

Sections MUST appear in this order when present.

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## Problem` | REQUIRED | Expanded problem statement. Markdown prose. |
| `## Solution` | REQUIRED | Expanded solution description. |
| `## Target Users` | REQUIRED | Detailed user personas. |
| `## Value Proposition` | REQUIRED | Why users choose this product. |
| `## Success Metrics` | OPTIONAL | Expanded metric definitions. |
| `## Competitive Landscape` | OPTIONAL | Detailed comparison. MAY include tables. |
| `## Pricing` | OPTIONAL | Pricing rationale and tier details. |
| `## Roadmap` | OPTIONAL | Future plans. No required structure. |

## Cross-References

- SHOULD reference `@GLOSSARY.md` for domain terms used in problem/solution.
- MAY reference `@BRAND.md#tagline` if tagline is shared.

## Validation Rules

1. `tagline` MUST be 120 characters or fewer.
2. `target_users` MUST contain at least 1 entry.
3. `value_propositions` MUST contain at least 1 entry.
4. Each `TargetUser.pain_points` MUST contain at least 1 entry.

## Conflict Detection

- `target_users` SHOULD match personas defined in `@BRAND.md` if both files exist.
- `tagline` SHOULD NOT contradict `@BRAND.md#tagline` if both are present.
