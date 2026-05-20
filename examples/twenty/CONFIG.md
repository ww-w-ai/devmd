---
devmd: config
version: "1.0"
project: Twenty CRM

config_sources:
  - type: environment_variables
    priority: 1
    description: "Primary configuration via .env or process.env"
  - type: workspace_settings
    priority: 2
    description: "Per-workspace overrides stored in DB"
  - type: feature_flags
    priority: 3
    description: "Per-workspace feature toggles via feature_flag table"
  - type: client_config
    priority: 4
    description: "Server config exposed to frontend via ClientConfig query"

categories:
  - database
  - redis
  - auth
  - email
  - storage
  - ai
  - billing
  - observability
  - server
  - frontend
---

# Twenty CRM — Configuration

All configuration for self-hosted and cloud deployments. Environment variables are the primary configuration mechanism. Workspace-specific settings override global defaults where applicable.

## Environment Variables

### Database

| Variable | Required | Default | Description |
|---|---|---|---|
| `PG_DATABASE_URL` | Yes | — | PostgreSQL connection string |
| `PG_SSL_ALLOW_SELF_SIGNED` | No | `false` | Allow self-signed SSL certificates |
| `PG_MAX_CONNECTIONS` | No | `20` | Connection pool maximum |
| `PG_IDLE_TIMEOUT_MS` | No | `30000` | Idle connection timeout |
| `PG_STATEMENT_TIMEOUT_MS` | No | `60000` | Query statement timeout |

See @SCHEMA.md#multi-tenant for how workspace schemas are managed within PostgreSQL.

### Redis

| Variable | Required | Default | Description |
|---|---|---|---|
| `REDIS_URL` | Yes | — | Redis connection string |
| `REDIS_TLS_ENABLED` | No | `false` | Enable TLS for Redis connection |
| `REDIS_MAX_RETRIES` | No | `3` | Connection retry attempts |

Redis is used for: BullMQ job queues, SSE pub/sub, cache, rate limiting. See @RUNTIME.md.

### ClickHouse

| Variable | Required | Default | Description |
|---|---|---|---|
| `CLICKHOUSE_URL` | No | — | ClickHouse connection URL (analytics, optional) |
| `CLICKHOUSE_USERNAME` | No | `default` | ClickHouse username |
| `CLICKHOUSE_PASSWORD` | No | — | ClickHouse password |

See @SCHEMA.md#clickhouse-analytics.

### Authentication Providers

| Variable | Required | Default | Description |
|---|---|---|---|
| `AUTH_PASSWORD_ENABLED` | No | `true` | Enable email+password login |
| `AUTH_GOOGLE_ENABLED` | No | `false` | Enable Google SSO |
| `AUTH_GOOGLE_CLIENT_ID` | If Google | — | Google OAuth client ID |
| `AUTH_GOOGLE_CLIENT_SECRET` | If Google | — | Google OAuth client secret |
| `AUTH_GOOGLE_CALLBACK_URL` | If Google | — | Google OAuth callback URL |
| `AUTH_MICROSOFT_ENABLED` | No | `false` | Enable Microsoft SSO |
| `AUTH_MICROSOFT_CLIENT_ID` | If MSFT | — | MSAL application client ID |
| `AUTH_MICROSOFT_CLIENT_SECRET` | If MSFT | — | MSAL client secret |
| `AUTH_MICROSOFT_TENANT_ID` | If MSFT | — | Azure AD tenant ID |
| `AUTH_MICROSOFT_CALLBACK_URL` | If MSFT | — | MSAL callback URL |
| `AUTH_SAML_ENABLED` | No | `false` | Enable SAML SSO |
| `AUTH_SAML_ISSUER` | If SAML | — | SAML issuer / entity ID |
| `AUTH_SAML_METADATA_URL` | If SAML | — | IdP metadata URL |
| `AUTH_SAML_CALLBACK_URL` | If SAML | — | ACS callback URL |
| `AUTH_OIDC_ENABLED` | No | `false` | Enable generic OIDC |
| `AUTH_OIDC_ISSUER_URL` | If OIDC | — | OIDC issuer discovery URL |
| `AUTH_OIDC_CLIENT_ID` | If OIDC | — | OIDC client ID |
| `AUTH_OIDC_CLIENT_SECRET` | If OIDC | — | OIDC client secret |
| `AUTH_OIDC_CALLBACK_URL` | If OIDC | — | OIDC callback URL |
| `AUTH_AZURE_AD_ENABLED` | No | `false` | Enable Azure AD (MSAL-specific) |
| `AUTH_AZURE_AD_TENANT_ID` | If Azure | — | Azure AD tenant ID |
| `AUTH_AZURE_AD_CLIENT_ID` | If Azure | — | Azure AD application ID |
| `AUTH_AZURE_AD_CLIENT_SECRET` | If Azure | — | Azure AD client secret |

See @SECURITY.md#authentication for auth flow details.

### Auth Provider Matrix

Which env vars each provider requires:

```
                    Password  Google  Microsoft  SAML   OIDC   Azure AD
─────────────────   ────────  ──────  ─────────  ────   ────   ────────
_ENABLED            ✓         ✓       ✓          ✓      ✓      ✓
_CLIENT_ID                    ✓       ✓                 ✓      ✓
_CLIENT_SECRET                ✓       ✓                 ✓      ✓
_CALLBACK_URL                 ✓       ✓          ✓      ✓
_TENANT_ID                            ✓                        ✓
_ISSUER                                          ✓
_METADATA_URL                                    ✓
_ISSUER_URL                                             ✓
```

### Token Configuration

| Variable | Required | Default | Description |
|---|---|---|---|
| `ACCESS_TOKEN_SECRET` | Yes | — | JWT signing secret for access tokens |
| `ACCESS_TOKEN_EXPIRY` | No | `15m` | Access token TTL |
| `REFRESH_TOKEN_SECRET` | Yes | — | JWT signing secret for refresh tokens |
| `REFRESH_TOKEN_EXPIRY` | No | `30d` | Refresh token TTL |
| `LOGIN_TOKEN_EXPIRY` | No | `10m` | Login token TTL (MFA flow) |

See @SECURITY.md#tokens.

### Email (SMTP for transactional emails)

| Variable | Required | Default | Description |
|---|---|---|---|
| `EMAIL_FROM_ADDRESS` | Yes | — | Sender address for system emails |
| `EMAIL_FROM_NAME` | No | `Twenty` | Sender name |
| `EMAIL_SMTP_HOST` | Yes | — | SMTP server hostname |
| `EMAIL_SMTP_PORT` | No | `587` | SMTP port |
| `EMAIL_SMTP_USER` | Yes | — | SMTP username |
| `EMAIL_SMTP_PASSWORD` | Yes | — | SMTP password |
| `EMAIL_SMTP_SECURE` | No | `false` | Use TLS for SMTP |

Note: These are for transactional emails (verification, password reset, invitations). Email sync uses connected account OAuth tokens. See @RUNTIME.md#email-calendar-sync.

### Storage

| Variable | Required | Default | Description |
|---|---|---|---|
| `STORAGE_TYPE` | No | `local` | `local` or `s3` |
| `STORAGE_LOCAL_PATH` | If local | `/app/storage` | Local filesystem path |
| `STORAGE_S3_REGION` | If S3 | — | AWS S3 region |
| `STORAGE_S3_NAME` | If S3 | — | S3 bucket name |
| `STORAGE_S3_ACCESS_KEY_ID` | If S3 | — | AWS access key |
| `STORAGE_S3_SECRET_ACCESS_KEY` | If S3 | — | AWS secret key |
| `STORAGE_S3_ENDPOINT` | No | — | Custom S3 endpoint (MinIO, R2) |

### AI Providers

| Variable | Required | Default | Description |
|---|---|---|---|
| `OPENAI_API_KEY` | No | — | OpenAI API key (global default) |
| `ANTHROPIC_API_KEY` | No | — | Anthropic API key (global default) |
| `GOOGLE_GENERATIVE_AI_API_KEY` | No | — | Google AI API key |
| `AZURE_OPENAI_API_KEY` | No | — | Azure OpenAI key |
| `AZURE_OPENAI_ENDPOINT` | No | — | Azure OpenAI endpoint URL |
| `AZURE_OPENAI_DEPLOYMENT` | No | — | Azure OpenAI deployment name |
| `AWS_REGION` | No | — | AWS Bedrock region |
| `AWS_ACCESS_KEY_ID` | No | — | AWS access key (Bedrock) |
| `AWS_SECRET_ACCESS_KEY` | No | — | AWS secret (Bedrock) |
| `MISTRAL_API_KEY` | No | — | Mistral API key |
| `XAI_API_KEY` | No | — | xAI (Grok) API key |

Note: Env vars set global defaults. Per-workspace AI settings override these. See @HARNESS.md.

### Billing

| Variable | Required | Default | Description |
|---|---|---|---|
| `BILLING_ENABLED` | No | `false` | Enable billing integration |
| `BILLING_STRIPE_SECRET_KEY` | If billing | — | Stripe secret key |
| `BILLING_STRIPE_WEBHOOK_SECRET` | If billing | — | Stripe webhook signing secret |
| `BILLING_FREE_TRIAL_DAYS` | No | `14` | Free trial duration |

### Observability

| Variable | Required | Default | Description |
|---|---|---|---|
| `SENTRY_DSN` | No | — | Sentry DSN (backend) |
| `REACT_APP_SENTRY_DSN` | No | — | Sentry DSN (frontend) |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | No | — | OpenTelemetry Collector endpoint |
| `OTEL_SERVICE_NAME` | No | `twenty-server` | Service name in traces |
| `LOG_LEVEL` | No | `INFO` | Minimum log level (DEBUG, INFO, WARN, ERROR) |
| `LOG_FORMAT` | No | `json` | Log format (`json` or `text`) |

See @LOGGING.md for trace propagation and metrics.

### Server

| Variable | Required | Default | Description |
|---|---|---|---|
| `SERVER_URL` | Yes | — | Backend URL (e.g., `https://api.twenty.com`) |
| `FRONT_BASE_URL` | Yes | — | Frontend URL (e.g., `https://app.twenty.com`) |
| `PORT` | No | `3000` | Server listen port |
| `NODE_ENV` | No | `production` | Environment mode |
| `RATE_LIMIT_MAX` | No | `100` | API rate limit (requests per second) |
| `RATE_LIMIT_BURST` | No | `200` | Rate limit burst allowance |
| `CORS_ORIGIN` | No | `FRONT_BASE_URL` | Allowed CORS origins |

### Frontend

| Variable | Required | Default | Description |
|---|---|---|---|
| `REACT_APP_SERVER_BASE_URL` | Yes | — | Backend API URL |
| `REACT_APP_VERSION` | No | `git SHA` | Application version for Sentry |

---

## Feature Flag System

Feature flags enable/disable features per workspace. Stored in the `feature_flag` table (public schema). See @SCHEMA.md#multi-tenant.

### Schema

```sql
CREATE TABLE feature_flag (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id UUID NOT NULL REFERENCES workspace(id),
  key VARCHAR(255) NOT NULL,
  value BOOLEAN NOT NULL DEFAULT false,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(workspace_id, key)
);
```

### Usage Pattern

```typescript
// Backend: check feature flag
const isEnabled = await featureFlagService.isEnabled(
  'IS_AI_ENABLED',
  workspaceId,
);

if (isEnabled) {
  // AI features available
}

// Frontend: received via ClientConfig query
const { data } = useQuery(GET_CLIENT_CONFIG);
if (data.clientConfig.featureFlags.isAiEnabled) {
  // Render AI UI elements
}
```

### Standard Feature Flags

| Flag Key | Default | Controls |
|---|---|---|
| `IS_AI_ENABLED` | `false` | AI features (chat, enrichment, agents) |
| `IS_WORKFLOW_ENABLED` | `true` | Workflow automation engine |
| `IS_CALENDAR_ENABLED` | `true` | Calendar sync |
| `IS_MESSAGING_ENABLED` | `true` | Email sync |
| `IS_BILLING_ENABLED` | `false` | Billing UI and enforcement |
| `IS_CUSTOM_OBJECTS_ENABLED` | `true` | Custom object creation |
| `IS_ANALYTICS_ENABLED` | `false` | ClickHouse analytics features |
| `IS_MULTI_WORKSPACE_ENABLED` | `false` | User can belong to multiple workspaces |

---

## ClientConfig — Server to Frontend

The frontend queries `clientConfig` at startup to receive server-side configuration. This avoids hardcoding backend settings in the frontend build.

```graphql
query GetClientConfig {
  clientConfig {
    authProviders {
      password
      google
      microsoft
      saml
      oidc
    }
    billing {
      enabled
      freeTrialDuration
    }
    telemetry {
      enabled
      anonymizationEnabled
    }
    support {
      enabled
      frontChatId
    }
    featureFlags {
      isAiEnabled
      isWorkflowEnabled
      isCalendarEnabled
      isMessagingEnabled
      isCustomObjectsEnabled
      isAnalyticsEnabled
    }
    frontDomain
    serverUrl
    defaultSubdomain
  }
}
```

### What ClientConfig Includes

| Category | Fields | Purpose |
|---|---|---|
| Auth providers | Which providers are enabled | Show/hide SSO buttons on login page |
| Billing | enabled, trial duration | Show billing UI, enforce limits |
| Feature flags | Boolean flags | Enable/disable UI sections |
| URLs | frontDomain, serverUrl | API routing, link generation |
| Telemetry | enabled, anonymization | User consent for analytics |

### What ClientConfig Does NOT Include

- API keys or secrets (never exposed to frontend)
- Database connection strings
- Internal service URLs
- Per-user permissions (fetched via separate `currentUser` query)

---

## Workspace Settings vs Global Settings

| Setting | Global (env var) | Workspace (DB) | Precedence |
|---|---|---|---|
| AI provider | Default provider | Per-workspace override | Workspace wins |
| Auth providers | Which are available | Cannot override (global only) | Global only |
| Storage type | local or S3 | Cannot override | Global only |
| Rate limits | Default limits | Cannot override | Global only |
| Feature flags | Default value | Per-workspace toggle | Workspace wins |
| Theme | System default | Per-user preference | User wins |
| Locale | Server default | Per-workspace + per-user | User > Workspace > Global |

---

## Configuration Validation

At startup, twenty-server validates all required environment variables. Missing required variables cause the server to exit with a clear error message listing what is missing.

```
[ERROR] Missing required environment variables:
  - PG_DATABASE_URL: PostgreSQL connection string
  - ACCESS_TOKEN_SECRET: JWT signing secret
  - SERVER_URL: Backend URL

See CONFIG.md for the full list of required variables.
```

Optional variables with invalid formats (e.g., non-numeric port) also cause startup failure with descriptive errors.

## Cross-References

- @SECURITY.md — Auth provider configuration, token settings
- @HARNESS.md — AI provider configuration per workspace
- @INFRA.md — Docker Compose and Kubernetes Helm values
- @RUNTIME.md — Redis and BullMQ queue settings
- @LOGGING.md — Observability configuration
- @SCHEMA.md — Database connection and multi-tenant settings
- @DEVOPS.md — Environment configuration in CI/CD pipelines
