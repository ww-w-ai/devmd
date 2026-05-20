---
devmd: devops
version: "1.0"
project: Twenty CRM

monorepo:
  tool: "Nx 21"
  packages: "20+"
  package_manager: "Yarn 4 (Berry)"

container:
  registry: "dockerhub (twentycrm)"
  images:
    - name: twentycrm/twenty-server
      entrypoints: [api, worker, cli]
    - name: twentycrm/twenty-front
      note: "Built into twenty-server image for self-hosted"

ci:
  platform: "GitHub Actions"
  workflow_count: "30+"
  strategy: per-package

cd:
  staging: "auto-deploy on push to main"
  production: "manual trigger on tag"

breaking_change_detection:
  graphql: graphql-inspector
  rest: openapi-diff
---

# Twenty CRM — DevOps

Build, test, deploy, and operate Twenty across environments.

## Docker Compose (Self-Hosted)

Four services compose the self-hosted deployment. See @INFRA.md for the full `docker-compose.yml`.

```
┌───────────────────────────────────────────────────┐
│ Docker Compose                                     │
│                                                   │
│  ┌──────────────┐     ┌─────────────────┐         │
│  │twenty-server │     │ twenty-worker   │         │
│  │ (API + Front)│     │ (BullMQ jobs)   │         │
│  │ port: 3000   │     │ no exposed port │         │
│  └──────┬───────┘     └────────┬────────┘         │
│         │                      │                  │
│         └──────┬───────────────┘                  │
│                │                                  │
│         ┌──────▼──────┐  ┌────────────┐           │
│         │ PostgreSQL  │  │   Redis    │           │
│         │ port: 5432  │  │ port: 6379 │           │
│         └─────────────┘  └────────────┘           │
└───────────────────────────────────────────────────┘
```

### Startup Order

1. **PostgreSQL** — healthcheck via `pg_isready` (10s interval, 5 retries)
2. **Redis** — starts immediately (no healthcheck dependency)
3. **twenty-server** — waits for PostgreSQL healthy, runs migrations on first start
4. **twenty-worker** — waits for PostgreSQL healthy, connects to Redis queues

### Volume Management

| Volume | Purpose | Backup Required |
|---|---|---|
| `pg_data` | PostgreSQL data directory | Yes — critical |
| `redis_data` | Redis persistence (RDB/AOF) | Recommended |
| `server_storage` | File attachments (local storage mode) | Yes — user files |

### Upgrade Procedure

```bash
# 1. Backup database
docker exec twenty-postgres pg_dump -U twenty twenty > backup_$(date +%F).sql

# 2. Pull new images
docker compose pull

# 3. Stop and restart (migrations run automatically)
docker compose down
docker compose up -d

# 4. Verify
docker compose logs twenty-server | grep "Migration"
curl -s http://localhost:3000/healthz
```

---

## Kubernetes Deployment

Production-grade deployment for Twenty Cloud and larger self-hosted installations. See @INFRA.md for the full cluster diagram and Helm values.

### Helm Chart Structure

```
twenty-helm/
├── Chart.yaml
├── values.yaml                    # Default values
├── values-staging.yaml            # Staging overrides
├── values-production.yaml         # Production overrides
├── templates/
│   ├── server-deployment.yaml     # API server pods
│   ├── server-service.yaml        # ClusterIP service
│   ├── worker-deployment.yaml     # Worker pods
│   ├── ingress.yaml               # Ingress with TLS
│   ├── postgres-statefulset.yaml  # PostgreSQL (or use managed)
│   ├── redis-statefulset.yaml     # Redis (or use managed)
│   ├── configmap.yaml             # Non-secret configuration
│   ├── secrets.yaml               # Secret references
│   ├── hpa.yaml                   # Horizontal Pod Autoscaler
│   ├── pdb.yaml                   # Pod Disruption Budget
│   ├── cronjob-backup.yaml        # Database backup CronJob
│   └── otel-collector.yaml        # OpenTelemetry Collector
└── tests/
    └── test-connection.yaml       # Helm test
```

### Key Helm Values

```yaml
# Server autoscaling
server:
  replicaCount: 2
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPU: 70
    targetMemory: 80
  resources:
    requests: { cpu: 500m, memory: 512Mi }
    limits: { cpu: 2000m, memory: 2Gi }
  readinessProbe:
    httpGet: { path: /healthz, port: 3000 }
    initialDelaySeconds: 10
    periodSeconds: 5
  livenessProbe:
    httpGet: { path: /healthz, port: 3000 }
    initialDelaySeconds: 30
    periodSeconds: 10

# Worker autoscaling (scales based on queue depth)
worker:
  replicaCount: 2
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 20
    metrics:
      - type: External
        external:
          metric: { name: bullmq_queue_waiting }
          target: { type: AverageValue, averageValue: 50 }

# Pod Disruption Budget
pdb:
  server: { minAvailable: 1 }
  worker: { minAvailable: 1 }
```

### Managed Services (Recommended for Production)

| Component | Self-Managed | Managed Alternative |
|---|---|---|
| PostgreSQL | StatefulSet | AWS RDS, GCP Cloud SQL, Supabase |
| Redis | StatefulSet | AWS ElastiCache, GCP Memorystore |
| ClickHouse | StatefulSet | ClickHouse Cloud |
| Object Storage | PVC | AWS S3, GCP Cloud Storage, Cloudflare R2 |

---

## CI/CD Pipeline — GitHub Actions

### Workflow Inventory (30+ workflows)

#### Per-Package CI

| Workflow | Trigger | Packages | Actions |
|---|---|---|---|
| `ci-twenty-front.yml` | PR + push to main | twenty-front | lint, type-check, unit tests, build |
| `ci-twenty-server.yml` | PR + push to main | twenty-server | lint, type-check, unit tests, integration tests |
| `ci-twenty-ui.yml` | PR + push to main | twenty-ui | lint, type-check, unit tests, Storybook build |
| `ci-twenty-shared.yml` | PR + push to main | twenty-shared | lint, type-check, unit tests |
| `ci-twenty-emails.yml` | PR + push to main | twenty-emails | lint, type-check, build |
| `ci-twenty-sdk.yml` | PR + push to main | twenty-sdk | lint, type-check, unit tests |

#### Cross-Package Workflows

| Workflow | Trigger | Purpose |
|---|---|---|
| `e2e-testing.yml` | PR | Playwright E2E tests (full stack) |
| `chromatic.yml` | PR | Visual regression (Storybook + Chromatic) |
| `breaking-change-detection.yml` | PR | GraphQL + OpenAPI schema diff |
| `bundle-size.yml` | PR | Frontend bundle size tracking |
| `i18n-check.yml` | PR | Missing translation key detection |
| `dependency-audit.yml` | Weekly cron | npm audit for vulnerabilities |
| `codeql-analysis.yml` | PR + weekly | GitHub CodeQL security scanning |
| `stale-issues.yml` | Daily cron | Close stale issues/PRs |

#### Build & Deploy Workflows

| Workflow | Trigger | Purpose |
|---|---|---|
| `docker-build.yml` | Push to main + tag | Build + push Docker images |
| `deploy-staging.yml` | Push to main | Deploy to staging via Helm |
| `deploy-production.yml` | Tag (v*) | Deploy to production via Helm |
| `deploy-docs.yml` | Push to main (docs changed) | Deploy documentation site |
| `release.yml` | Tag (v*) | GitHub Release + changelog |

### CI Environment Setup

```yaml
# Shared setup steps across CI workflows
- uses: actions/checkout@v4
- uses: actions/setup-node@v4
  with:
    node-version: 24
    cache: yarn
- run: yarn install --immutable
- run: npx nx affected --target=lint --base=origin/main
- run: npx nx affected --target=test --base=origin/main
```

### Nx Affected — Smart Test Selection

Nx tracks dependencies between packages. On PR, only affected packages are tested:

```bash
# Only lint/test packages affected by changes in this PR
npx nx affected --target=lint --base=origin/main
npx nx affected --target=test --base=origin/main
npx nx affected --target=build --base=origin/main
```

This reduces CI time from ~30 minutes (all packages) to ~5-10 minutes (affected only).

---

## Breaking Change Detection

Runs on every PR. Blocks merge if breaking changes are detected without a migration plan.

### GraphQL Schema Diff

```yaml
# .github/workflows/breaking-change-detection.yml
- name: Generate schema from main
  run: |
    git checkout origin/main
    npx nx run twenty-server:schema:export -- --output schema-main.graphql

- name: Generate schema from PR
  run: |
    git checkout ${{ github.head_ref }}
    npx nx run twenty-server:schema:export -- --output schema-pr.graphql

- name: Diff schemas
  run: |
    npx graphql-inspector diff \
      schema-main.graphql \
      schema-pr.graphql \
      --rule suppressRemovalOfDeprecatedField
```

### OpenAPI Diff

```yaml
- name: Diff REST API schemas
  run: |
    npx openapi-diff openapi-main.json openapi-pr.json
```

### What Blocks Merge

| Change Type | Blocks? | Mitigation |
|---|---|---|
| Field removed | Yes | Deprecate first, remove in next major |
| Type changed (breaking) | Yes | Add new field, deprecate old |
| Enum value removed | Yes | Deprecate first |
| Argument made required | Yes | Make optional with default |
| Field added | No | Non-breaking |
| Enum value added | No | Non-breaking |
| Field deprecated | No | Allowed with migration timeline |

See @TESTING.md#breaking-change-detection, @CHANGELOG.md.

---

## Nx Build Caching

### Local Cache

Nx caches build outputs locally in `node_modules/.cache/nx`. Repeated builds of unchanged packages are instant.

### Remote Cache (CI)

```yaml
# nx.json (relevant section)
{
  "tasksRunnerOptions": {
    "default": {
      "runner": "nx-cloud",
      "options": {
        "accessToken": "${{ secrets.NX_CLOUD_ACCESS_TOKEN }}",
        "cacheableOperations": ["build", "lint", "test", "e2e"]
      }
    }
  }
}
```

Benefits:
- Developer A builds `twenty-ui` → cache uploaded to Nx Cloud.
- Developer B pulls same branch → `twenty-ui` build served from cache (0 seconds).
- CI uses same cache → reduced build times across workflows.

---

## Grafana + OpenTelemetry Setup

See @LOGGING.md for metrics and trace details.

### Docker Compose (Observability Stack)

```yaml
# docker-compose.observability.yml (optional, extend main compose)
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
    volumes:
      - ./otel-config.yaml:/etc/otel/config.yaml
    command: ["--config=/etc/otel/config.yaml"]

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3002:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources

  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - loki_data:/loki

  tempo:
    image: grafana/tempo:latest
    ports:
      - "3200:3200"
    volumes:
      - tempo_data:/var/tempo

volumes:
  grafana_data:
  loki_data:
  tempo_data:
```

### OTel Collector Configuration

```yaml
# otel-config.yaml
receivers:
  otlp:
    protocols:
      grpc: { endpoint: "0.0.0.0:4317" }
      http: { endpoint: "0.0.0.0:4318" }

processors:
  batch:
    timeout: 5s
    send_batch_size: 1000

exporters:
  loki:
    endpoint: "http://loki:3100/loki/api/v1/push"
  otlp/tempo:
    endpoint: "tempo:4317"
    tls: { insecure: true }
  prometheus:
    endpoint: "0.0.0.0:8889"

service:
  pipelines:
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [loki]
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/tempo]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
```

---

## Development Environment

```bash
# Prerequisites
node >= 24
yarn >= 4
docker (for PostgreSQL + Redis)

# Setup
git clone https://github.com/twentyhq/twenty.git
cd twenty
yarn install
cp .env.example .env  # Edit with local values

# Start services
docker compose up -d postgres redis
nx serve twenty-server
nx serve twenty-front

# Run all checks (pre-commit)
npx tsgo                              # Type check (Rust-based, fast)
nx run-many --target=lint --all       # Lint
nx run-many --target=test --all       # Unit + integration tests
```

See @CLAUDE.md#development-workflow for coding conventions.

## Cross-References

- @INFRA.md — Infrastructure topology, Docker Compose, Kubernetes
- @TESTING.md — Test pyramid, CI test stages, visual regression
- @LOGGING.md — OpenTelemetry integration, Grafana dashboards
- @CONFIG.md — Environment variables for all services
- @SECURITY.md — CI security scanning (CodeQL, dependency audit)
- @CHANGELOG.md — Breaking change tracking in releases
- @CLAUDE.md — Development workflow and coding conventions
