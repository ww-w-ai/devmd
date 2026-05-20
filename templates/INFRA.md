---
devmd: infra
version: 0.1.0

provider: ""                     # cloudflare | aws | gcp | vercel | ...

compute:
  - name: ""
    type: ""                     # workers | lambda | container | vm | edge
    runtime: ""
    region: ""
    scaling: ""                  # auto | fixed | ...

database:
  - name: ""
    type: ""                     # d1 | rds | planetscale | supabase | ...
    region: ""
    backup: ""                   # daily | continuous | ...

cache:
  type: ""                       # kv | redis | memcached | none
  ttl: ""

cdn:
  provider: ""
  custom_domain: ""
  ssl: ""                        # auto | custom

dns:
  provider: ""
  domain: ""
  records: []

secrets:
  manager: ""                    # ref @SECURITY.md#secrets
  env_file: ""                   # ref @CONFIG.md

monitoring:
  apm: ""
  logging: ""                    # ref @LOGGING.md
  alerting: ""
---

# INFRA.md

> Infrastructure intent — provider, compute, database, cache, DNS, monitoring.

## Overview

<!-- High-level infra diagram. Reference @ARCHITECTURE.md for system structure. -->

## Compute

<!-- Runtime environments. Reference @DEVOPS.md#deploy-targets for deployment. -->

## Database

<!-- DB instances, backup strategy. Reference @SCHEMA.md for schema. -->

## Cache

<!-- Cache layer, TTL rules. Reference @API.md for cacheable endpoints. -->

## CDN & DNS

<!-- Domain, SSL, CDN config. Reference @SEO.md#cache-strategy. -->

## Monitoring

<!-- APM, logging, alerts. Reference @LOGGING.md and @OPERATIONS.md#health-checks. -->

## Cost Estimate

<!-- Monthly cost breakdown by service. -->

| Service | Tier | Est. Monthly |
|---------|------|-------------|
|         |      |             |
