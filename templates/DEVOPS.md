---
devmd: devops
version: 0.1.0

docker:
  base_image: ""
  dockerfile: ""
  compose: false

ci_cd:
  provider: ""                   # github-actions | gitlab-ci | circleci | ...
  workflows:
    - name: ""
      trigger: ""                # push | pr | tag | schedule
      steps: []

build:
  command: ""
  output_dir: ""
  env: ""                        # ref @CONFIG.md

deploy_targets:
  - name: ""                     # production | staging | preview
    provider: ""                 # ref @INFRA.md#compute
    url: ""
    branch: ""
    auto_deploy: false

rollback:
  strategy: ""                   # revert-commit | blue-green | canary-rollback
  max_rollback_versions: 0
---

# DEVOPS.md

> Docker, CI/CD workflows, build pipeline, deploy targets, and rollback strategy.

## Docker

<!-- Base image, Dockerfile, compose setup. Reference @INFRA.md#compute. -->

## CI/CD Workflows

<!-- One subsection per workflow. Reference @TESTING.md#ci for test pipeline. -->

### [Workflow Name]

- **Trigger:**
- **Steps:**
  1.
  2.
  3.

## Build Pipeline

<!-- Build command, output, env. Reference @CONFIG.md for build-time vars. -->

## Deploy Targets

| Target | Provider | URL | Branch | Auto |
|--------|----------|-----|--------|------|
|        |          |     |        |      |

## Rollback Strategy

<!-- How to roll back. Reference @OPERATIONS.md#runbooks for incident response. -->

## Branch Strategy

<!-- Main, develop, feature branches. Reference @CHANGELOG.md for release flow. -->
