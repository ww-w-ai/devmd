---
devmd: security
version: 0.1.0

auth:
  method: ""                     # jwt | session | oauth2 | api-key
  provider: ""                   # self | auth0 | supabase | clerk | ...
  mfa: false
  token:
    access_ttl: ""
    refresh_ttl: ""
    storage: ""                  # httponly-cookie | memory | localStorage

rbac:
  roles: []                      # e.g. [admin, editor, viewer]
  default_role: ""
  permission_model: ""           # role-based | attribute-based | policy-based

owasp:
  injection: ""                  # parameterized queries | ORM | ...
  xss: ""                        # CSP | sanitize | escape
  csrf: ""                       # token | same-site | ...
  auth_broken: ""                # mitigation

csp:
  default_src: ""
  script_src: ""
  style_src: ""
  img_src: ""

rate_limiting:
  enabled: true
  strategy: ""                   # ref @API.md#rate-limits

secrets:
  manager: ""                    # env | vault | aws-sm | ...
  rotation_policy: ""
---

# SECURITY.md

> Authentication, RBAC, OWASP checklist, CSP, rate limiting, and secrets management.

## Authentication

<!-- Auth method, provider, token lifecycle. Reference @API.md#authentication. -->

## RBAC

<!-- Roles, permissions. Reference @SCHEMA.md for user/role tables. -->

## OWASP Top 10

<!-- Mitigation for each relevant item. -->

| Threat | Mitigation | Status |
|--------|-----------|--------|
| Injection | | |
| XSS | | |
| CSRF | | |
| Broken Auth | | |

## Content Security Policy

<!-- CSP directives. Reference @INFRA.md for header config. -->

## Secrets Management

<!-- Where secrets live, rotation. Reference @CONFIG.md#env-vars. -->

## Vulnerability Reporting

<!-- How to report security issues. -->
