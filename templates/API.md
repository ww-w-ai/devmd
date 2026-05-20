---
devmd: api
version: 0.1.0

api:
  style: ""                      # REST | GraphQL | gRPC | tRPC
  base_url: ""
  versioning: ""                 # url-prefix | header | query-param
  auth: ""                       # bearer | api-key | cookie | none

endpoints:
  - method: ""                   # GET | POST | PUT | PATCH | DELETE
    path: ""
    summary: ""
    auth_required: true
    request:
      params: []
      body: ""                   # schema ref or inline
    response:
      success: ""                # status code + schema
      errors: []                 # ref @ERRORS.md codes

pagination:
  style: ""                      # cursor | offset | page
  default_limit: 20
  max_limit: 100

rate_limits:
  - tier: ""
    requests_per_minute: 0
---

# API.md

> Endpoints, authentication, error format, versioning, and rate limits.

## Base Configuration

<!-- API style, base URL, versioning strategy. Reference @ARCHITECTURE.md#layers. -->

## Authentication

<!-- Auth method, token format, refresh flow. Reference @SECURITY.md#auth. -->

## Endpoints

<!-- One subsection per resource group. -->

### [Resource]

| Method | Path | Summary | Auth | Ref |
|--------|------|---------|------|-----|
|        |      |         |      |     |

## Error Format

<!-- Standard error response shape. Reference @ERRORS.md for full code list. -->

```json
{
  "error": { "code": "", "message": "", "details": [] }
}
```

## Pagination

<!-- Style, params, response envelope. -->

## Rate Limits

<!-- Tiers and limits. Reference @OPERATIONS.md#slos for SLA. -->
