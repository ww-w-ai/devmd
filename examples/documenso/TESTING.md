---
devmd: testing
version: "1.0"
project: documenso
updated: 2026-05-13

frameworks:
  e2e: playwright
  linting: biome
  type_checking: typescript-strict
  commit_linting: commitlint
  security_scanning: codeql
  git_hooks: husky

ci:
  platform: github-actions
  total_workflows: 16
  e2e_runner: 8-core
  e2e_timeout: 60min

test_directories: 22
coverage_tool: none  # No unit test coverage tool; E2E is primary strategy
---

# TESTING.md — Documenso

## Testing Strategy

Documenso's testing strategy is **E2E-first**. The primary test suite is Playwright browser tests that exercise the full application stack (database, API, UI). This reflects the nature of the product: document signing is a workflow-heavy domain where integration matters more than unit isolation.

```
Testing Pyramid (Documenso's actual shape):

          ┌──────────┐
          │  Manual   │  ← PR review, design review
          │  QA       │
       ┌──┴──────────┴──┐
       │   Playwright     │  ← PRIMARY: 22 test directories
       │   E2E Tests      │
    ┌──┴──────────────────┴──┐
    │   Type Checking          │  ← TypeScript strict across all packages
    │   + Biome Linting        │
    └──────────────────────────┘
```

## Playwright E2E Tests

Located in `packages/app-tests/e2e/`.

### Test Directories (22)

| Directory | Domain | Key Scenarios |
|---|---|---|
| `auth/` | Authentication | Sign up, sign in, password reset, email verification |
| `auth-2fa/` | Two-Factor Auth | TOTP enable/disable, login with 2FA |
| `auth-passkey/` | WebAuthn | Passkey registration, login with passkey |
| `document/` | Document CRUD | Create, edit, delete, duplicate documents |
| `document-send/` | Sending | Send document, resend, cancel |
| `document-sign/` | Signing Flow | All 11 field types, signature capture |
| `document-complete/` | Completion | Document sealing, download signed PDF |
| `document-reject/` | Rejection | Recipient rejects, status transitions |
| `template/` | Templates | Create, edit, use templates |
| `template-direct-link/` | Direct Links | Public signing via direct links |
| `recipient/` | Recipients | Add, remove, reorder, role assignment |
| `field/` | Fields | Place fields, configure options, validation |
| `team/` | Teams | Create team, invite members, switch teams |
| `organisation/` | Organisations | Org settings, member management |
| `profile/` | User Profile | Update profile, change password, delete account |
| `folder/` | Folders | Create, move documents to folders |
| `api-token/` | API Tokens | Generate, revoke API tokens |
| `webhook/` | Webhooks | Register, test webhooks |
| `admin/` | Admin Panel | User management, platform stats |
| `embedding/` | Embedding | Embedded signing flow |
| `i18n/` | Internationalization | Language switching, RTL support |
| `signing-order/` | Sequential Signing | Ordered recipient signing flow |

### Test Configuration

```typescript
// playwright.config.ts
export default defineConfig({
  testDir: './e2e',
  timeout: 60_000,          // 60s per test
  expect: { timeout: 10_000 },
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 4 : undefined,
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
  ],
  webServer: {
    command: 'npm run dev',
    port: 3000,
    reuseExistingServer: !process.env.CI,
  },
});
```

### Test Helpers

```typescript
// Common test utilities
import { seedUser } from './helpers/seed-user';
import { seedDocument } from './helpers/seed-document';
import { seedTeam } from './helpers/seed-team';
import { apiLogin } from './helpers/api-login';

// Seeding creates fresh test data per test
// API login bypasses UI for faster setup
// Each test gets isolated database state
```

## CI Pipeline

### GitHub Actions Workflows (16)

| Workflow | Trigger | Purpose |
|---|---|---|
| `ci.yml` | PR, push to main | Lint + type-check + E2E tests |
| `e2e.yml` | PR | Full Playwright suite (8-core runner, 60min) |
| `lint.yml` | PR | Biome lint check |
| `typecheck.yml` | PR | TypeScript compilation |
| `codeql.yml` | Schedule + PR | Security vulnerability scanning |
| `deploy-preview.yml` | PR | Deploy preview environment |
| `deploy-production.yml` | Push to main | Production deployment |
| `docker-build.yml` | Release | Build and push Docker images |
| `translations.yml` | PR | Verify i18n catalogs |
| `pr-auto-label.yml` | PR | Auto-label PRs by changed files |
| `pr-title-check.yml` | PR | Verify PR title follows commitlint |
| `stale.yml` | Schedule | Close stale issues/PRs |
| `release.yml` | Manual | Create release with changelog |
| `dependabot-auto.yml` | Dependabot PR | Auto-merge minor/patch updates |
| `lock-threads.yml` | Schedule | Lock old resolved threads |
| `contributor-welcome.yml` | First PR | Welcome new contributors |

### E2E CI Configuration

```yaml
# .github/workflows/e2e.yml
jobs:
  e2e:
    runs-on: ubuntu-latest-8-cores  # 8-core runner
    timeout-minutes: 60
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: documenso_test
          POSTGRES_USER: documenso
          POSTGRES_PASSWORD: documenso
        ports:
          - 5432:5432
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - run: npm ci
      - run: npx prisma migrate deploy
      - run: npx playwright install chromium
      - run: npx playwright test
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: packages/app-tests/playwright-report/
```

## Linting

### Biome (Not ESLint)

Documenso uses [Biome](https://biomejs.dev/) for both linting and formatting. No ESLint or Prettier.

```json
// biome.json (root)
{
  "formatter": {
    "indentStyle": "tab",
    "indentWidth": 2,
    "lineWidth": 100
  },
  "linter": {
    "rules": {
      "recommended": true,
      "suspicious": {
        "noExplicitAny": "warn"
      }
    }
  },
  "organizeImports": {
    "enabled": true
  }
}
```

### Git Hooks (Husky)

```
pre-commit:
  - lint-staged (runs Biome on staged files)
  - TypeScript type-check on changed packages

commit-msg:
  - commitlint (conventional commits)
```

### Commitlint

```
type(scope): description

Types: feat, fix, chore, docs, style, refactor, perf, test, ci, build
Scopes: app, lib, trpc, api, prisma, ui, email, auth, signing, ee, docs
```

## CodeQL Security Scanning

Runs on schedule and PRs. Scans for:

- SQL injection patterns
- XSS vulnerabilities
- Path traversal
- Hardcoded credentials
- Insecure cryptographic usage

## Testing Conventions

1. **One test file per feature flow**. Name: `{feature}.spec.ts`
2. **Seed data per test**. No shared state between tests. Each test creates its own users, documents, etc.
3. **API login for setup**. Use API helpers to log in and seed, save UI login for auth-specific tests.
4. **Locator strategy**: Prefer `data-testid` attributes, fall back to accessible roles.
5. **No unit tests required** for PRs. E2E coverage is the quality bar.
6. **Screenshots on failure** are uploaded as CI artifacts.
7. **Traces on first retry** for debugging flaky tests.

## Cross-References

- CI/CD infrastructure: @INFRA.md#ci-cd
- Error types tested: @ERRORS.md
- Security scanning: @SECURITY.md#codeql
- All 11 field types tested: @GLOSSARY.md#field-type-reference
