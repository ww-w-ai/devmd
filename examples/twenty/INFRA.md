---
devmd: infra
version: "1.0"
project: Twenty CRM

intent:
  deployment: "Self-hosted (Docker Compose) or managed (Twenty Cloud)"
  scaling: "Horizontal scaling via Kubernetes for multi-tenant SaaS"
  provider_agnostic: true

docker_compose:
  services:
    - name: twenty-server
      image: "twentycrm/twenty-server:latest"
      ports: ["3000:3000"]
      entry_points: [api, worker]
      env:
        - PG_DATABASE_URL
        - REDIS_URL
        - STORAGE_TYPE (local | s3)
        - FRONT_BASE_URL
        - SERVER_URL
        - SENTRY_DSN
        - SIGN_IN_PREFILLED (dev only)
      depends_on: [postgres, redis]
    - name: twenty-worker
      image: "twentycrm/twenty-server:latest"
      command: "node dist/src/main worker"
      env: "same as twenty-server"
      depends_on: [postgres, redis]
    - name: postgres
      image: "postgres:16"
      ports: ["5432:5432"]
      volumes: ["pg_data:/var/lib/postgresql/data"]
      env:
        - POSTGRES_USER
        - POSTGRES_PASSWORD
        - POSTGRES_DB
    - name: redis
      image: "redis:7"
      ports: ["6379:6379"]
      volumes: ["redis_data:/data"]
  volumes:
    - pg_data
    - redis_data
    - server_local_storage

kubernetes:
  tool: "Helm charts + raw K8s manifests"
  components:
    - name: twenty-server
      type: Deployment
      replicas: "2+ (HPA based on CPU/request rate)"
      resources:
        requests: { cpu: "500m", memory: "512Mi" }
        limits: { cpu: "2000m", memory: "2Gi" }
    - name: twenty-worker
      type: Deployment
      replicas: "2+ (HPA based on queue depth)"
      resources:
        requests: { cpu: "250m", memory: "256Mi" }
        limits: { cpu: "1000m", memory: "1Gi" }
    - name: postgres
      type: StatefulSet
      replicas: 1
      storage: "PersistentVolumeClaim (100Gi+)"
      backup: "pg_dump daily + WAL archiving"
    - name: redis
      type: StatefulSet
      replicas: 1
      storage: "PersistentVolumeClaim (10Gi)"
    - name: ingress
      type: Ingress
      tls: "cert-manager (Let's Encrypt)"
      annotations: "nginx.ingress.kubernetes.io/*"
  autoscaling:
    server:
      min: 2
      max: 10
      metric: "cpu > 70% OR request_rate > 100/s"
    worker:
      min: 2
      max: 20
      metric: "queue_depth > 100 OR job_wait_time > 30s"

observability:
  collector: OpenTelemetry Collector
  metrics: Grafana
  traces: "Grafana Tempo"
  logs: "Grafana Loki"
  errors: Sentry 10
  dashboards:
    - "API Overview (request rate, latency, errors)"
    - "Database (query latency, connections, slow queries)"
    - "Worker (job throughput, queue depth, failures)"
    - "Email Sync (messages synced, errors, latency)"
    - "Business Metrics (users, records, API calls)"
  alerts:
    - name: high_error_rate
      condition: "error_rate > 5% for 5 minutes"
      severity: critical
      channel: slack
    - name: high_latency
      condition: "p95_latency > 2s for 10 minutes"
      severity: warning
      channel: slack
    - name: db_connection_exhaustion
      condition: "active_connections > 80% of max"
      severity: critical
      channel: pagerduty
    - name: worker_queue_backlog
      condition: "queue_depth > 1000 for 15 minutes"
      severity: warning
      channel: slack

storage:
  options:
    - type: local
      path: "/app/storage"
      use_case: "Self-hosted, single server"
    - type: s3
      bucket: configurable
      region: configurable
      use_case: "Multi-server, cloud deployments"
    - type: cloudflare_r2
      use_case: "Edge-optimized storage"

ci_cd:
  platform: GitHub Actions
  workflow_count: "30+"
  strategy: "Per-package isolation — each package has its own CI"
  pipelines:
    - name: lint-and-type-check
      trigger: "PR to main"
      steps: [checkout, install, oxlint, tsgo]
    - name: unit-tests
      trigger: "PR to main"
      steps: [checkout, install, "nx run-many --target=test"]
      matrix: "[twenty-front, twenty-server, twenty-ui, twenty-shared]"
    - name: integration-tests
      trigger: "PR to main"
      services: [postgres, redis]
      steps: [checkout, install, migrate, "nx test twenty-server --integration"]
    - name: e2e-tests
      trigger: "PR to main"
      services: [postgres, redis]
      steps: [checkout, install, build, deploy-preview, "nx e2e twenty-e2e-testing"]
    - name: visual-regression
      trigger: "PR to main"
      steps: [checkout, install, "nx build-storybook twenty-ui", chromatic-publish]
    - name: breaking-change-detection
      trigger: "PR to main"
      steps: [checkout, install, generate-schema, graphql-diff, openapi-diff]
    - name: docker-build
      trigger: "Push to main"
      steps: [checkout, docker-build, docker-push-to-registry]
    - name: deploy-staging
      trigger: "Push to main"
      steps: [docker-pull, helm-upgrade-staging, smoke-test]
    - name: deploy-production
      trigger: "Manual or tag release"
      steps: [docker-pull, helm-upgrade-production, smoke-test, rollback-on-failure]

multi_tenant_infra:
  strategy: "schema-per-workspace in shared PostgreSQL"
  workspace_provisioning:
    - Create PostgreSQL schema (workspace_{id})
    - Create standard object tables
    - Seed default metadata (fields, views, pipeline stages)
    - Configure workspace settings
  workspace_deletion:
    - Soft delete workspace record
    - Background job drops workspace schema after grace period (30 days)
  scaling_concern: >
    At 10,000+ workspaces, schema count becomes a PostgreSQL concern.
    Future: shard workspaces across multiple PostgreSQL instances.
---

# Twenty CRM — Infrastructure

## Self-Hosted Deployment (Docker Compose)

The simplest deployment for self-hosting. Four containers, one `docker-compose.yml`.

```yaml
# docker-compose.yml
version: '3.9'
services:
  twenty-server:
    image: twentycrm/twenty-server:latest
    ports:
      - "3000:3000"
    environment:
      PG_DATABASE_URL: postgres://twenty:twenty@postgres:5432/twenty
      REDIS_URL: redis://redis:6379
      FRONT_BASE_URL: http://localhost:3001
      SERVER_URL: http://localhost:3000
      STORAGE_TYPE: local
      STORAGE_LOCAL_PATH: /app/storage
    volumes:
      - server_storage:/app/storage
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started

  twenty-worker:
    image: twentycrm/twenty-server:latest
    command: ["node", "dist/src/main", "worker"]
    environment:
      PG_DATABASE_URL: postgres://twenty:twenty@postgres:5432/twenty
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started

  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: twenty
      POSTGRES_PASSWORD: twenty
      POSTGRES_DB: twenty
    volumes:
      - pg_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U twenty"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7
    volumes:
      - redis_data:/data

volumes:
  pg_data:
  redis_data:
  server_storage:
```

## Kubernetes Deployment

For production SaaS (Twenty Cloud) and larger self-hosted deployments.

```
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                     │
│                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │
│  │ Ingress     │  │ Cert-Manager│  │ OTel Collector  │  │
│  │ (nginx)     │  │ (TLS)       │  │                 │  │
│  └──────┬──────┘  └─────────────┘  └────────┬────────┘  │
│         │                                    │           │
│  ┌──────▼──────┐  ┌─────────────┐           │           │
│  │twenty-server│  │twenty-worker│           │           │
│  │ Deployment  │  │ Deployment  │    ┌──────▼──────┐   │
│  │ 2-10 pods   │  │ 2-20 pods   │    │  Grafana    │   │
│  │ (HPA)       │  │ (HPA)       │    │  Loki       │   │
│  └──────┬──────┘  └──────┬──────┘    │  Tempo      │   │
│         │                │           └─────────────┘   │
│  ┌──────▼────────────────▼──────┐                      │
│  │      Service Mesh             │                      │
│  └──────┬────────────────┬──────┘                      │
│         │                │                              │
│  ┌──────▼──────┐  ┌─────▼─────┐                       │
│  │ PostgreSQL  │  │   Redis   │                       │
│  │ StatefulSet │  │ StatefulSet│                       │
│  │ + PVC       │  │ + PVC     │                       │
│  └─────────────┘  └───────────┘                       │
└─────────────────────────────────────────────────────────┘
```

### Helm Values (Key Configuration)

```yaml
# values.yaml (abbreviated)
server:
  replicaCount: 2
  image:
    repository: twentycrm/twenty-server
    tag: latest
  resources:
    requests: { cpu: 500m, memory: 512Mi }
    limits: { cpu: 2000m, memory: 2Gi }
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilization: 70

worker:
  replicaCount: 2
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 20

postgres:
  storage: 100Gi
  backup:
    enabled: true
    schedule: "0 3 * * *"  # Daily at 3 AM

redis:
  storage: 10Gi

ingress:
  enabled: true
  tls: true
  certManager: true
```

## CI/CD Pipeline

30+ GitHub Actions workflows with per-package isolation.

```
PR to main
  ├── Lint (Oxlint + Prettier) ──── blocking
  ├── Type Check (tsgo) ─────────── blocking
  ├── Unit Tests (Jest + Vitest) ── blocking
  ├── Integration Tests (Jest) ──── blocking (needs PG + Redis)
  ├── E2E Tests (Playwright) ────── blocking (needs full stack)
  ├── Visual Regression ─────────── non-blocking (Chromatic)
  ├── Breaking Change Detection ─── blocking (schema diff)
  └── Bundle Size Check ─────────── non-blocking (warn on +10%)

Push to main
  ├── Docker Build + Push ─── twentycrm/twenty-server:latest
  ├── Deploy Staging ──────── Helm upgrade → smoke test
  └── Notify Slack

Tag Release (v1.x.x)
  ├── Docker Build + Push ─── twentycrm/twenty-server:v1.x.x
  ├── Deploy Production ───── Helm upgrade → smoke test → rollback on failure
  ├── GitHub Release ──────── Changelog + artifacts
  └── Notify community
```

## Database Operations

### Backup Strategy

| Method | Frequency | Retention | Use Case |
|---|---|---|---|
| pg_dump (logical) | Daily 3 AM UTC | 30 days | Point-in-time restore |
| WAL archiving | Continuous | 7 days | PITR to any second |
| Schema export | On every metadata change | 90 days | Schema-level restore |

### Migration Strategy

Migrations apply to two levels:
1. **Public schema** (core tables) — Standard TypeORM migrations, versioned in code
2. **Workspace schemas** — Metadata-driven, applied dynamically when workspace connects

See @SCHEMA.md#multi-tenant for details.

## Network Security

- TLS 1.3 termination at Ingress (cert-manager + Let's Encrypt)
- Internal traffic: mTLS optional (service mesh)
- PostgreSQL: TLS required, password auth, no public exposure
- Redis: Password auth, no public exposure
- API rate limiting: 100 req/s default, 200 burst. See @API.md#rate-limits

## Cross-References

- @ARCHITECTURE.md — System components that map to infrastructure
- @SCHEMA.md — Multi-tenant PostgreSQL schema strategy
- @LOGGING.md — OpenTelemetry and Grafana integration
- @SECURITY.md — Network and data encryption
- @TESTING.md — CI pipeline test stages
- @RUNTIME.md — Worker and cron job infrastructure
