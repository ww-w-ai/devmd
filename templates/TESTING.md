---
devmd: testing
version: 0.1.0

pyramid:
  unit:
    framework: ""                # vitest | jest | pytest | ...
    coverage_target: 0           # percentage
    location: ""                 # e.g. "__tests__/" or "*.test.ts"
  integration:
    framework: ""
    coverage_target: 0
  e2e:
    framework: ""                # playwright | cypress | ...
    coverage_target: 0
    critical_paths: []           # flows from @FLOWS.md

ci:
  trigger: ""                    # push | pr | merge
  pipeline: ""                   # ref @DEVOPS.md#ci-cd
  fail_on:
    - coverage_below: 0
    - lint_errors: true
    - type_errors: true

linting:
  tools: []                      # eslint | biome | ruff | ...
  config_path: ""

mocking:
  strategy: ""                   # msw | nock | factory | ...
  fixtures_path: ""
---

# TESTING.md

> Test pyramid, frameworks, CI pipeline, coverage targets, and linting.

## Test Pyramid

<!-- Unit → Integration → E2E ratio. Reference @ARCHITECTURE.md#layers for boundaries. -->

## Unit Tests

<!-- Framework, location, patterns. Reference @SCHEMA.md for model tests. -->

## Integration Tests

<!-- API testing, DB testing. Reference @API.md#endpoints for test targets. -->

## E2E Tests

<!-- Critical paths. Reference @FLOWS.md for flow coverage. -->

## CI Pipeline

<!-- When tests run, fail conditions. Reference @DEVOPS.md#ci-cd. -->

## Linting & Formatting

<!-- Tools, config. Reference @CLAUDE.md for AI-enforced rules. -->

## Mocking Strategy

<!-- How external dependencies are mocked. -->
