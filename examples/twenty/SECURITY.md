---
devmd: security
version: "1.0"
project: Twenty CRM
contact: security@twenty.com
disclosure: responsible
pgp_key: null

authentication:
  providers:
    - type: password
      enforcement: "Regex-based strength validation"
      hashing: bcrypt
    - type: google_sso
      protocol: OAuth 2.0
      scopes: [email, profile]
    - type: microsoft_sso
      protocol: OAuth 2.0
      library: MSAL
    - type: saml
      library: node-saml
      metadata_url: configurable
    - type: oidc
      library: openid-client
      discovery: "/.well-known/openid-configuration"
    - type: azure_ad
      library: MSAL
      tenant: configurable
  mfa:
    type: TOTP
    library: "otpauth (RFC 6238)"
    backup_codes: true
    enforcement: "optional (admin can require)"

tokens:
  - type: access_token
    format: JWT
    expiry: 15min
    payload: [userId, workspaceId, role, permissions]
  - type: refresh_token
    format: JWT
    expiry: 30d
    storage: httpOnly cookie
    rotation: true
  - type: login_token
    format: JWT
    expiry: 10min
    purpose: "Short-lived token for login flow completion"
  - type: workspace_agnostic_token
    format: JWT
    expiry: 5min
    purpose: "Cross-workspace operations (workspace selection)"
  - type: email_verification_token
    format: JWT
    expiry: 24h
    purpose: "Email address verification"
  - type: api_key
    format: "app_token (random string, hashed in DB)"
    expiry: configurable
    purpose: "Programmatic API access"

rbac:
  layers:
    - layer: workspace_role
      description: "Owner, Admin, Member, Guest"
      granularity: workspace-wide
    - layer: settings_permissions
      description: "WORKSPACE, ROLES, SECURITY, DATA_MODEL, WORKFLOWS, AI_SETTINGS"
      granularity: settings-category
    - layer: object_permissions
      description: "CREATE, READ, UPDATE, DELETE per object type"
      granularity: per-object-type
    - layer: field_permissions
      description: "READ, UPDATE per field on an object"
      granularity: per-field
    - layer: row_level_security
      description: "WHERE predicates applied at query time based on user role"
      granularity: per-record
  enforcement:
    - point: WorkspaceAuthGuard
      level: "JWT validation + workspace context"
    - point: SettingsPermissionGuard
      level: "Settings-level permission check"
    - point: WorkspaceEntityManager
      level: "ORM-level object + field + row permission enforcement"
    - point: GraphQL field resolver
      level: "Field-level visibility filtering"

custom_roles:
  enabled: true
  description: >
    Admins can create custom roles with specific permission sets. Roles
    are defined as a collection of permission flags. Users can be assigned
    one role per workspace.
---

# Twenty CRM Security

## Authentication

### Multi-Provider Auth Flow

```
User visits /login
  ├── Email + Password → validate credentials → issue tokens
  ├── Google SSO → OAuth redirect → callback → match/create user → issue tokens
  ├── Microsoft SSO → MSAL flow → callback → match/create user → issue tokens
  ├── SAML → IdP redirect → ACS callback → match/create user → issue tokens
  └── OIDC → Discovery → auth redirect → callback → match/create user → issue tokens

All paths → Access Token (15min) + Refresh Token (30d, httpOnly cookie)
```

### Password Policy

- Minimum 8 characters
- Regex-based validation (configurable by workspace admin)
- Bcrypt hashing (cost factor 10)
- No password history enforcement (not yet implemented)
- Password reset via email verification token

### Two-Factor Authentication (TOTP)

1. User enables 2FA in settings
2. Server generates TOTP secret and QR code
3. User scans with authenticator app (Google Authenticator, Authy, etc.)
4. User enters verification code to confirm setup
5. Backup codes generated (10 codes, single-use)
6. On login: email+password → MFA challenge → TOTP code → tokens issued

### Token Lifecycle

```
Login
  → Login Token (10min, completes MFA if required)
  → Access Token (15min, used for all API calls)
  → Refresh Token (30d, httpOnly cookie, rotated on each refresh)

Token Refresh (when access token expires)
  → Send refresh token to /auth/refresh
  → Old refresh token revoked
  → New access + refresh tokens issued
  → If refresh token expired → full re-authentication

Workspace Selection
  → Workspace Agnostic Token (5min, for listing/selecting workspaces)
  → User selects workspace
  → Full access + refresh tokens issued for that workspace
```

## RBAC — Four Permission Layers

See @GLOSSARY.md#rbac for term definitions.

### Layer 1: Workspace Role

| Role | Description | Scope |
|---|---|---|
| Owner | Full control, can delete workspace | All operations |
| Admin | Manage settings, members, data model | All except billing/delete workspace |
| Member | Standard CRM operations | CRUD on permitted objects |
| Guest | Read-only access to shared views | Read-only on permitted objects |

### Layer 2: Settings Permissions

Fine-grained permissions for workspace settings. Assigned to roles.

| Permission | Controls |
|---|---|
| WORKSPACE | Workspace name, logo, domain settings |
| ROLES | Create/edit/delete custom roles and permission sets |
| SECURITY | Auth providers, SSO config, 2FA enforcement, API keys |
| DATA_MODEL | Create/edit/delete objects, fields, relations, indexes |
| WORKFLOWS | Create/edit/delete workflow automations |
| AI_SETTINGS | AI provider config, skill permissions, agent config |

### Layer 3: Object-Level Permissions

Per-object CRUD permissions assigned to roles.

```
Role: "Sales Rep"
  Person: CREATE, READ, UPDATE
  Company: READ
  Opportunity: CREATE, READ, UPDATE, DELETE
  Note: CREATE, READ, UPDATE, DELETE
  Task: CREATE, READ, UPDATE, DELETE
  Dashboard: READ
  Settings: (none)
```

### Layer 4: Row-Level Security

WHERE predicates injected at query time by WorkspaceEntityManager.

```typescript
// Example: Sales reps can only see opportunities they own
{
  role: 'sales_rep',
  object: 'opportunity',
  predicate: 'assigneeId = :currentUserId'
}

// Example: Guests can only see records in shared views
{
  role: 'guest',
  object: '*',
  predicate: 'id IN (SELECT recordId FROM view_record WHERE viewId IN (SELECT id FROM view WHERE isShared = true))'
}
```

### Permission Enforcement Points

```
GraphQL Request
  │
  ▼
WorkspaceAuthGuard
  │ Validates JWT, extracts workspace context
  │ Rejects: expired/invalid tokens
  ▼
SettingsPermissionGuard (settings endpoints only)
  │ Checks settings-level permissions
  │ Rejects: insufficient settings permissions
  ▼
GraphQL Resolver
  │ Calls service layer
  ▼
WorkspaceEntityManager (twenty-orm)
  │ Checks object-level permissions (can user access this object?)
  │ Checks field-level permissions (filters response fields)
  │ Applies row-level security predicates (adds WHERE clauses)
  │ Rejects: permission denied at any level
  ▼
Database Query (workspace_{id} schema)
```

## SQL Injection Prevention

- All queries go through twenty-orm parameterized query builders. See @ARCHITECTURE.md#twenty-orm
- Raw SQL is prohibited (enforced by `no-raw-sql` Oxlint rule). See @TESTING.md
- User input in filters is parameterized, never interpolated
- Custom field names validated against metadata (only known fields allowed)
- Database user has minimal privileges (no DDL except during migration)

## Data Encryption

| Data | At Rest | In Transit |
|---|---|---|
| Database | PostgreSQL TDE (optional) | TLS 1.3 |
| OAuth tokens | AES-256-GCM encrypted columns | TLS 1.3 |
| API keys | bcrypt hashed (only hash stored) | TLS 1.3 |
| File uploads | Filesystem/S3 encryption | TLS 1.3 |
| Redis | Not encrypted (ephemeral cache) | TLS optional |

## Security Headers

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Referrer-Policy: strict-origin-when-cross-origin
```

## Vulnerability Disclosure

Report security issues to security@twenty.com. Responsible disclosure policy: 90-day window before public disclosure.

## Cross-References

- @ERRORS.md — Authentication and authorization error codes
- @API.md — Auth headers and token usage in API calls
- @ARCHITECTURE.md — Guard middleware pipeline
- @LOGGING.md — Security event logging
- @INFRA.md — Network-level security (TLS, firewalls)
