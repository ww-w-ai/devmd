# SECURITY.md Specification

> Version: 0.1.0-draft | Status: Draft

## Purpose

SECURITY.md defines authentication methods, authorization policies, and security posture. It extends the GitHub SECURITY.md convention (vulnerability reporting) with structured, machine-readable security configuration for AI-driven development.

## Frontmatter Schema

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Project name |
| `auth_methods` | `AuthMethod[]` | REQUIRED | Supported authentication methods. Min 1 entry. |
| `token_types` | `TokenType[]` | OPTIONAL | Token definitions for auth and API access |
| `rbac` | `RBAC` | OPTIONAL | Role-based access control configuration |
| `owasp` | `map<string, enum(mitigated\|partial\|not-applicable)>` | OPTIONAL | OWASP Top 10 compliance status |
| `csp` | `CSP` | OPTIONAL | Content Security Policy configuration |
| `rate_limiting` | `RateLimiting` | OPTIONAL | Rate limiting configuration |
| `secrets` | `Secrets` | OPTIONAL | Secrets management policy |

### AuthMethod

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `type` | `enum(password\|oauth\|saml\|oidc\|api-key\|passkey\|2fa)` | REQUIRED | Authentication type |
| `provider` | `string` | REQUIRED | Provider name (e.g., `"google"`, `"auth0"`, `"internal"`) |
| `required` | `boolean` | OPTIONAL | Whether this method is mandatory. Default: `false`. |

### TokenType

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Token identifier (e.g., `"access_token"`, `"refresh_token"`) |
| `purpose` | `string` | REQUIRED | What this token is used for |
| `ttl` | `string` | REQUIRED | Time to live (e.g., `"15m"`, `"7d"`, `"30d"`) |
| `storage` | `enum(httponly-cookie\|memory\|localStorage\|sessionStorage\|header)` | REQUIRED | Client-side storage mechanism |

### RBAC

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `roles` | `map<string, Role>` | OPTIONAL | Named role definitions |
| `levels` | `enum(role\|object\|field\|row)[]` | OPTIONAL | Granularity levels of access control |

### Role

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `description` | `string` | REQUIRED | Role purpose |
| `permissions` | `string[]` | REQUIRED | Permission identifiers. Min 1 entry. |

### CSP

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `enabled` | `boolean` | REQUIRED | Whether CSP is active |
| `directives` | `map<string, string>` | OPTIONAL | CSP directive to value map (e.g., `"default-src": "'self'"`) |

### RateLimiting

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `enabled` | `boolean` | REQUIRED | Whether rate limiting is active |
| `tiers` | `RateTier[]` | OPTIONAL | Rate limit tiers |

### RateTier

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `scope` | `string` | REQUIRED | What is limited (e.g., `"ip"`, `"user"`, `"api-key"`) |
| `requests` | `number` | REQUIRED | Max requests allowed |
| `window` | `string` | REQUIRED | Time window (e.g., `"1m"`, `"1h"`, `"1d"`) |

### Secrets

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `manager` | `string` | REQUIRED | Secrets manager (e.g., `"vault"`, `"aws-secrets-manager"`, `"env"`, `"1password"`) |
| `rotation` | `string` | OPTIONAL | Rotation policy (e.g., `"90d"`, `"manual"`, `"on-compromise"`) |

## Body Sections

Sections MUST appear in this order when present.

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## Authentication` | REQUIRED | Auth flow descriptions, provider setup, and MFA policy. |
| `## Authorization / RBAC` | REQUIRED | Role hierarchy, permission model, and enforcement points. |
| `## OWASP Compliance` | OPTIONAL | Per-item status of OWASP Top 10 mitigation. |
| `## Content Security Policy` | OPTIONAL | CSP directives and rationale. |
| `## Rate Limiting` | OPTIONAL | Rate limit tiers, bypass rules, and response codes. |
| `## Secrets Management` | OPTIONAL | Secret storage, rotation, and access patterns. |
| `## Vulnerability Reporting` | OPTIONAL | Contact info and disclosure policy. Follows GitHub SECURITY.md convention. |

## Cross-References

- MUST reference `@API.md` for authentication headers and protected endpoints.
- SHOULD reference `@SCHEMA.md` for user, role, and permission data models.
- SHOULD reference `@INFRA.md` for secrets manager and network security configuration.
- SHOULD reference `@ERRORS.md` for authentication and authorization error codes.

## Validation Rules

1. `auth_methods` MUST contain at least 1 entry.
2. Every `AuthMethod` MUST include `type` and `provider`.
3. Every `TokenType` MUST include `name`, `purpose`, `ttl`, and `storage`.
4. `TokenType.storage` value `localStorage` SHOULD trigger a security warning (XSS risk).
5. Every `Role.permissions` MUST contain at least 1 entry.
6. `owasp` keys SHOULD use standard identifiers (e.g., `"A01:2021-Broken Access Control"`).
7. `rate_limiting.tiers` SHOULD be present when `rate_limiting.enabled` is `true`.

## Conflict Detection

- `auth_methods` MUST match the authentication described in `@API.md#authentication` if both files exist.
- `rbac.roles` SHOULD align with user/role models in `@SCHEMA.md`.
- `secrets.manager` SHOULD match the secrets infrastructure in `@INFRA.md`.
- `token_types` with `storage: httponly-cookie` MUST have corresponding `Set-Cookie` behavior in `@API.md`.
