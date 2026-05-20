---
devmd: security
version: "1.0"
project: documenso
updated: 2026-05-13

certifications: [SOC2]
license: AGPL-3.0

authentication:
  methods:
    - email_password
    - oauth_google
    - oauth_oidc
    - webauthn_passkeys
    - totp_2fa
  session:
    type: encrypted_cookie
    name: __session
    encryption: server-side
  identity_providers: [DOCUMENSO, GOOGLE, OIDC]

authorization:
  model: RBAC
  levels:
    - scope: organisation
      roles: [ADMIN, MANAGER, MEMBER]
    - scope: team
      roles: [ADMIN, MANAGER, MEMBER]

signing:
  providers:
    - name: local
      method: "PKCS#12 (P12) certificate"
      use_case: "Self-hosted, development"
    - name: gcloud-hsm
      method: "Google Cloud HSM"
      use_case: "Production, compliance-critical"
---

# SECURITY.md — Documenso

## Authentication

### Auth Server Architecture

Documenso runs a **custom auth server** built on Hono (`@documenso/auth`). It is mounted at `/auth` on the main Hono server. 9 route files handle all auth flows.

```
/auth
├── /signin          POST — Email/password login
├── /signup          POST — Registration
├── /signout         POST — Session destruction
├── /verify-email    POST — Email verification token
├── /forgot-password POST — Password reset request
├── /reset-password  POST — Password reset execution
├── /oauth/callback  GET  — OAuth callback handler
├── /passkey         POST — WebAuthn assertion
└── /2fa/verify      POST — TOTP verification
```

### Email/Password

- Passwords hashed with **bcrypt** (cost factor 12)
- Email verification required before full access
- Password reset via time-limited tokens (1 hour expiry)

### OAuth

OAuth providers implemented via **Arctic** library (lightweight OAuth 2.0 client):

| Provider | Protocol | Identity Provider Enum |
|---|---|---|
| Google | OAuth 2.0 | `GOOGLE` |
| Custom OIDC | OpenID Connect | `OIDC` |

OAuth flow:
1. Redirect to provider authorization URL
2. Provider redirects back with code
3. Exchange code for tokens
4. Create/link user account
5. Create session

### WebAuthn / Passkeys

- Registration via `@simplewebauthn/server`
- Credential stored: `credentialId`, `credentialPublicKey`, `counter`, `deviceType`, `backedUp`
- Supports cross-platform authenticators (USB keys, phone, platform biometrics)
- See @SCHEMA.md#passkey for database model

### Two-Factor Authentication (TOTP)

- Standard TOTP (RFC 6238) with 30-second window
- Secret stored encrypted in `User.twoFactorSecret`
- Backup codes provided at setup (one-time use)
- Required on every login when enabled

### Session Management

```typescript
// Session lifecycle
{
  id: string,       // cuid
  userId: number,
  token: string,     // random token, encrypted in cookie
  expiresAt: Date,   // 30 days from creation
  createdAt: Date
}
```

- Session token is **encrypted server-side** before setting as `__session` cookie
- `HttpOnly`, `Secure`, `SameSite=Lax`
- CSRF protection via **double-submit cookie** pattern + **origin validation**
- Session invalidation on password change

## Authorization (RBAC)

### Organisation Level

| Role | Capabilities |
|---|---|
| **ADMIN** | Full org management, billing, member management, all team access |
| **MANAGER** | Team management, member invites, document management |
| **MEMBER** | Document creation and management within assigned teams |

### Team Level

| Role | Capabilities |
|---|---|
| **ADMIN** | Team settings, member management, API tokens, webhooks |
| **MANAGER** | Document management, template management |
| **MEMBER** | Create and sign documents, use templates |

### Permission Resolution

```
User accesses resource →
  1. Is user member of the team owning the resource?
  2. Does user's team role permit the operation?
  3. For org-wide operations: check org member role
  4. API tokens: inherit team-level permissions of the creating user
```

## PDF Cryptographic Signing

### Signing Providers

#### Local (PKCS#12)

- Self-hosted default
- P12 certificate file loaded from filesystem or environment variable
- Signs PDF with embedded digital signature
- Certificate subject/issuer stored in DocumentMeta

#### Google Cloud HSM

- Production-grade hardware security module
- Private key never leaves HSM
- FIPS 140-2 Level 3 certified
- Used by Documenso cloud platform

### Sealing Process

```
1. All recipients complete their fields
2. System triggers seal operation
3. Collect all signature data from fields
4. Render signatures onto PDF pages
5. Generate cryptographic hash of document
6. Sign hash with certificate (local P12 or HSM)
7. Embed digital signature into PDF (PAdES format)
8. Store sealed PDF via storage provider
9. Update document status to COMPLETED
10. Notify all recipients with sealed document
```

See @GLOSSARY.md#seal for domain definition.

## Content Security Policy (CSP)

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'nonce-{per-request-nonce}';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: blob:;
  font-src 'self';
  connect-src 'self' https://*.posthog.com;
  frame-ancestors 'self' {configured-embed-origins};
  base-uri 'self';
  form-action 'self';
```

- **Per-request nonces** for inline scripts (no `unsafe-inline` for scripts)
- `frame-ancestors` configurable for embedding use case
- CSP report-uri for violation monitoring

## Rate Limiting

- **Backend**: Database-backed sliding window
- **Strategy**: Per IP address + per authenticated user
- **Endpoints**:

| Endpoint Category | Limit |
|---|---|
| Auth (login, signup) | 10 requests/minute per IP |
| API (authenticated) | 100 requests/minute per user |
| Signing (recipient actions) | 30 requests/minute per token |
| File upload | 10 requests/minute per user |
| Password reset | 3 requests/hour per email |

- Returns `429 Too Many Requests` with `Retry-After` header
- See @ERRORS.md#too_many_requests

## Captcha

- **Cloudflare Turnstile** on public-facing forms (signup, password reset, direct link signing)
- Server-side token verification
- Configurable via environment variable `NEXT_PUBLIC_TURNSTILE_SITE_KEY`

## Audit Trail

18 event types tracked in `DocumentAuditLog` (see @LOGGING.md#audit-event-types):

- Every event records: timestamp, user ID, IP address, user agent
- Audit log is **append-only** (no updates or deletes)
- Exportable as PDF for compliance
- Stored alongside the document for legal validity

## Data Protection

### At Rest

- PostgreSQL with filesystem-level encryption (provider-dependent)
- Sensitive fields (2FA secret, OAuth tokens) encrypted at application level
- PDF documents stored via configurable provider (DB or S3 with server-side encryption)

### In Transit

- TLS 1.2+ required for all connections
- HSTS headers enforced
- Database connections use SSL in production

### Data Deletion

- User account deletion removes personal data
- Document deletion is soft-delete (audit trail preserved)
- GDPR data export available via profile settings

## Vulnerability Reporting

Security issues should be reported to `security@documenso.com`.

- Do not open public GitHub issues for security vulnerabilities
- Response SLA: 48 hours for initial acknowledgment
- Responsible disclosure timeline: 90 days

## Cross-References

- Auth database models: @SCHEMA.md#auth-models
- Auth API routes: @API.md#auth
- Error codes for auth: @ERRORS.md#authentication-errors
- CSP nonce in UI: @UI.md#security-headers
- Audit log format: @LOGGING.md#audit-events
