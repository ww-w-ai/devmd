# INFRA.md Specification

> Version: 0.1.0-draft | Status: Draft

## Purpose

INFRA.md declares infrastructure **intent** (WHAT), not implementation (HOW). It is runtime-agnostic: an AI or human reads INFRA.md and generates the appropriate artifact — `wrangler.toml`, Terraform, `docker-compose.yml`, or any other format. Terraform is HOW (code); INFRA.md is WHAT (intent).

## Frontmatter Schema

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Infrastructure configuration name |
| `provider` | `enum(aws\|gcp\|azure\|cloudflare\|vercel\|railway\|self-hosted\|...)` | REQUIRED | Primary cloud provider |
| `tier` | `enum(starter\|production\|enterprise)` | OPTIONAL | Infrastructure complexity tier |
| `compute` | `Compute` | REQUIRED | Compute runtime specification |
| `database` | `Database` | REQUIRED | Primary data store |
| `cache` | `Cache` | OPTIONAL | Caching layer |
| `queue` | `Queue` | OPTIONAL | Message queue or job broker |
| `storage` | `Storage` | OPTIONAL | Object or file storage |
| `cdn` | `CDN` | OPTIONAL | Content delivery |
| `dns` | `DNS` | OPTIONAL | Domain configuration |
| `secrets` | `string[]` | REQUIRED | Secret names (MUST NOT contain values) |
| `monitoring` | `Monitoring` | OPTIONAL | Observability stack |

### Compute

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `runtime` | `string` | REQUIRED | Runtime identifier (e.g., `node20`, `python3.12`, `workers`) |
| `plan` | `string` | OPTIONAL | Provider plan or instance type |
| `regions` | `string[]` | REQUIRED | Deployment regions. Min 1. |

### Database

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `type` | `string` | REQUIRED | Database engine (e.g., `postgresql`, `d1`, `dynamodb`) |
| `version` | `string` | OPTIONAL | Engine version |
| `backup` | `string` | OPTIONAL | Backup strategy (e.g., `daily`, `continuous`) |
| `replicas` | `number` | OPTIONAL | Read replica count |

### Cache

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `type` | `string` | REQUIRED | Cache engine (e.g., `redis`, `memcached`, `kv`) |
| `strategy` | `enum(write-through\|write-behind\|cache-aside\|read-through)` | OPTIONAL | Caching strategy |
| `ttl` | `string` | OPTIONAL | Default time-to-live |

### Queue

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `type` | `string` | REQUIRED | Queue system (e.g., `sqs`, `rabbitmq`, `bullmq`) |
| `provider` | `string` | OPTIONAL | Managed provider if different from primary |

### Storage

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `type` | `enum(object\|block\|file)` | REQUIRED | Storage class |
| `provider` | `string` | REQUIRED | Storage provider (e.g., `s3`, `r2`, `gcs`) |
| `bucket` | `string` | OPTIONAL | Default bucket name |

### CDN

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `provider` | `string` | REQUIRED | CDN provider |
| `domains` | `string[]` | REQUIRED | Domains served. Min 1. |

### DNS

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `domain` | `string` | REQUIRED | Primary domain |
| `proxy` | `boolean` | OPTIONAL | Whether traffic is proxied (e.g., Cloudflare orange-cloud) |

### Monitoring

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `provider` | `string` | REQUIRED | Monitoring platform |
| `alerts` | `string[]` | OPTIONAL | Alert channel identifiers |

## Body Sections

Sections MUST appear in this order when present.

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## Compute` | REQUIRED | Runtime details, scaling rules, cold-start considerations. |
| `## Database` | REQUIRED | Schema references, connection pooling, failover. |
| `## Cache` | OPTIONAL | Invalidation strategy, key naming. |
| `## Queue` | OPTIONAL | Queue topology, dead-letter policy. |
| `## Storage` | OPTIONAL | Access patterns, lifecycle rules. |
| `## DNS & CDN` | OPTIONAL | Domain mapping, TLS, cache rules. |
| `## Monitoring` | OPTIONAL | Dashboards, alert thresholds. |
| `## Constraints` | OPTIONAL | Budget limits, compliance requirements, data residency. |

## Cross-References

- MUST reference `@SCHEMA.md` for database entity alignment.
- MUST reference `@CONFIG.md` for environment variable mapping.
- SHOULD reference `@SECURITY.md` for secret management policy.

## Validation Rules

1. `secrets` MUST NOT contain actual secret values — only names.
2. `compute.regions` MUST contain at least 1 entry.
3. Every secret name SHOULD match an env var of type `secret` in `@CONFIG.md`.
4. `database.type` MUST be a recognized engine identifier.

## Conflict Detection

- Every secret in `secrets` SHOULD have a corresponding entry in `@CONFIG.md#variables`.
- `provider` SHOULD be consistent with `@DEVOPS.md#deploy.targets` providers.
- `database.type` MUST NOT contradict `@SCHEMA.md#engine` if both are present.
