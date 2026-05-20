---
devmd: config
version: 0.1.0

env_vars:
  - name: ""
    required: true
    default: null
    description: ""
    sensitive: false              # if true, must use @SECURITY.md#secrets
    scope: ""                     # all | production | development | test

provider_matrix:
  - provider: ""                 # cloudflare | aws | vercel | ...
    env_mapping: {}              # provider-specific env var names

feature_flags:
  system: ""                     # env | launchdarkly | custom | none
  flags:
    - name: ""
      default: false
      description: ""
      rollout: ""                # percentage | user-list | all
---

# CONFIG.md

> Environment variables, provider matrix, and feature flags.

## Environment Variables

<!-- One row per var. Reference @SECURITY.md#secrets for sensitive values. -->

| Name | Required | Default | Description | Sensitive |
|------|----------|---------|-------------|-----------|
|      |          |         |             |           |

## Provider Matrix

<!-- Map env vars to provider-specific names. Reference @INFRA.md#provider. -->

## Feature Flags

<!-- Flag definitions and rollout strategy. -->

| Flag | Default | Description | Rollout |
|------|---------|-------------|---------|
|      |         |             |         |

## Config Validation

<!-- How config is validated at startup. Reference @ERRORS.md for invalid config errors. -->

## Cross-References

- Secrets: @SECURITY.md#secrets
- Infrastructure: @INFRA.md
- Deploy: @DEVOPS.md
