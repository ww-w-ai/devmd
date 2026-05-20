# CONFIG.md Specification

> Version: 0.1.0-draft | Status: Draft

## Purpose

CONFIG.md declares all environment variables, provider selection matrices, and feature flags. It is the single source of truth for runtime configuration. Secrets are named here; values live in the environment or a vault.

## Frontmatter Schema

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Configuration set name |
| `variables` | `map<string, EnvVar>` | REQUIRED | Environment variables keyed by name |
| `providers` | `map<string, ProviderMatrix>` | OPTIONAL | Swappable provider selections |
| `feature_flags` | `map<string, FeatureFlag>` | OPTIONAL | Feature toggles |

### EnvVar

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `required` | `boolean` | REQUIRED | Whether the app fails without this variable |
| `default` | `any` | OPTIONAL | Default value. MUST NOT be set for type `secret`. |
| `type` | `enum(string\|number\|boolean\|url\|secret)` | REQUIRED | Value type |
| `description` | `string` | REQUIRED | Human-readable purpose |
| `category` | `string` | OPTIONAL | Grouping label (e.g., `database`, `auth`, `api`) |

### ProviderMatrix

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `options` | `string[]` | REQUIRED | Available provider choices. Min 1. |
| `env_var` | `string` | REQUIRED | Env var that selects the active provider |
| `default` | `string` | OPTIONAL | Default provider if env var is unset |

### FeatureFlag

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `description` | `string` | REQUIRED | What this flag controls |
| `default` | `boolean` | REQUIRED | Default state |

## Body Sections

Sections MUST appear in this order when present.

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## Environment Variables` | REQUIRED | Table or list of all variables with descriptions. |
| `## Provider Matrix` | OPTIONAL | Explanation of swappable providers and their trade-offs. |
| `## Feature Flags` | OPTIONAL | Flag descriptions and rollout criteria. |
| `## Valid Combinations` | OPTIONAL | Known conflicts or incompatibilities between provider choices. |
| `## Development Defaults` | OPTIONAL | Recommended `.env.example` values for local development. |

## Cross-References

- MUST reference `@INFRA.md` for infrastructure-level config alignment.
- SHOULD reference `@SECURITY.md` for variables of type `secret`.
- SHOULD reference `@RUNTIME.md` for job provider configuration.

## Validation Rules

1. Every variable with `required: true` and no `default` MUST be documented as required in the body.
2. Variables of type `secret` MUST NOT have a `default` value.
3. Every `ProviderMatrix.env_var` MUST exist in `variables`.
4. `ProviderMatrix.default` MUST be one of `ProviderMatrix.options`.
5. `ProviderMatrix.options` MUST contain at least 1 entry.

## Conflict Detection

- Every env var referenced in `@INFRA.md#secrets` MUST be defined in `variables` with type `secret`.
- Every env var referenced by `@RUNTIME.md` or `@DEVOPS.md` SHOULD exist in `variables`.
- If `@INFRA.md#provider` is set, at least one `ProviderMatrix` SHOULD align with it.
