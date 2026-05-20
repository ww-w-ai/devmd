# DEVOPS.md Specification

> Version: 0.1.0-draft | Status: Draft

## Purpose

DEVOPS.md defines CI/CD pipelines, build configuration, deployment targets, and backup procedures. It bridges the gap between infrastructure intent (`@INFRA.md`) and operational practice (`@OPERATIONS.md`).

## Frontmatter Schema

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | DevOps configuration name |
| `docker` | `Docker` | OPTIONAL | Container configuration |
| `ci` | `CI` | REQUIRED | Continuous integration setup |
| `build` | `Build` | OPTIONAL | Build toolchain |
| `deploy` | `Deploy` | OPTIONAL | Deployment configuration |
| `backup` | `Backup` | OPTIONAL | Backup and recovery policy |

### Docker

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `base_image` | `string` | REQUIRED | Base Docker image (e.g., `node:20-alpine`) |
| `stages` | `string[]` | OPTIONAL | Multi-stage build stage names |
| `ports` | `number[]` | OPTIONAL | Exposed ports |

### CI

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `provider` | `enum(github-actions\|gitlab-ci\|circleci\|jenkins\|...)` | REQUIRED | CI platform |
| `workflows` | `map<string, Workflow>` | REQUIRED | Named workflow definitions. Min 1. |

### Workflow

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `trigger` | `string` | REQUIRED | Event that starts this workflow (e.g., `push:main`, `pull_request`) |
| `description` | `string` | REQUIRED | What this workflow does |
| `critical` | `boolean` | OPTIONAL | If `true`, merge MUST NOT proceed when this workflow fails |

### Build

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `tool` | `string` | REQUIRED | Build tool (e.g., `vite`, `turbopack`, `webpack`, `esbuild`) |
| `cache_strategy` | `string` | OPTIONAL | Build cache approach |

### Deploy

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `targets` | `map<string, DeployTarget>` | OPTIONAL | Named deploy environments |
| `strategy` | `enum(rolling\|blue-green\|canary\|recreate)` | OPTIONAL | Default deployment strategy |

### DeployTarget

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `provider` | `string` | REQUIRED | Deploy platform (e.g., `cloudflare`, `vercel`, `aws-ecs`) |
| `url` | `url` | OPTIONAL | Target environment URL |
| `auto_deploy` | `boolean` | OPTIONAL | Whether deploy triggers automatically on CI success |

### Backup

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `schedule` | `string` | REQUIRED | Backup frequency (e.g., `daily`, `hourly`, cron expression) |
| `retention` | `string` | REQUIRED | How long backups are kept (e.g., `30d`, `1y`) |
| `provider` | `string` | OPTIONAL | Backup storage provider |

## Body Sections

Sections MUST appear in this order when present.

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## Docker` | OPTIONAL | Dockerfile rationale, layer optimization notes. |
| `## CI/CD Workflows` | REQUIRED | Detailed workflow descriptions, required checks. |
| `## Build Pipeline` | OPTIONAL | Build steps, environment requirements, caching. |
| `## Deploy Targets` | OPTIONAL | Environment-specific deploy instructions. |
| `## Backup & Recovery` | OPTIONAL | Backup verification, restore procedures. |
| `## Rollback Procedure` | OPTIONAL | Step-by-step rollback instructions per deploy target. |

## Cross-References

- MUST reference `@INFRA.md` for provider and compute alignment.
- SHOULD reference `@TESTING.md` for CI test requirements.
- SHOULD reference `@CONFIG.md` for env vars used in CI/CD pipelines.

## Validation Rules

1. `ci.workflows` MUST contain at least 1 workflow.
2. Every workflow with `critical: true` MUST have a `trigger` that includes merge-blocking events.
3. `deploy.targets` providers SHOULD be consistent with `@INFRA.md#provider`.
4. `backup.schedule` MUST be a recognized frequency or valid cron expression.

## Conflict Detection

- `deploy.targets` providers MUST NOT contradict `@INFRA.md#provider` without explicit justification.
- If `@TESTING.md` defines required test suites, at least one CI workflow SHOULD run them.
- `docker.ports` SHOULD match ports declared in `@INFRA.md#compute` if applicable.
