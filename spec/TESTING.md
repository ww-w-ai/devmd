# TESTING.md Specification

> Version: 0.1.0-draft | Status: Draft

## Purpose

TESTING.md defines the test strategy — test pyramid, frameworks, coverage targets, CI integration, and test data management. It provides a declarative specification for how a project validates correctness.

## Frontmatter Schema

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Project name |
| `frameworks` | `map<string, Framework>` | REQUIRED | Test frameworks keyed by name. Min 1 entry. |
| `pyramid` | `Pyramid` | OPTIONAL | Test distribution targets |
| `coverage` | `Coverage` | OPTIONAL | Code coverage requirements |
| `ci` | `CI` | OPTIONAL | CI pipeline integration |

### Framework

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `type` | `enum(unit\|integration\|e2e\|visual\|lint\|performance\|security)` | REQUIRED | Test category |
| `package` | `string` | REQUIRED | Package or tool name (e.g., `"vitest"`, `"playwright"`, `"eslint"`) |
| `config` | `string` | OPTIONAL | Config file path relative to project root |

### Pyramid

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `unit` | `number` | REQUIRED | Unit test percentage target (0-100) |
| `integration` | `number` | REQUIRED | Integration test percentage target (0-100) |
| `e2e` | `number` | REQUIRED | E2E test percentage target (0-100) |

### Coverage

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `target` | `number` | REQUIRED | Coverage percentage target (0-100) |
| `enforced` | `boolean` | OPTIONAL | Whether CI fails below target. Default: `false`. |
| `exclude` | `string[]` | OPTIONAL | Glob patterns excluded from coverage |

### CI

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `provider` | `string` | REQUIRED | CI service (e.g., `"github-actions"`, `"gitlab-ci"`, `"circleci"`) |
| `workflows` | `string[]` | REQUIRED | Workflow file paths. Min 1 entry. |
| `test_on` | `string[]` | OPTIONAL | Trigger events (e.g., `["push", "pull_request"]`) |

## Body Sections

Sections MUST appear in this order when present.

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## Test Pyramid` | REQUIRED | Distribution rationale and target percentages. |
| `## Frameworks` | REQUIRED | Per-framework setup, configuration, and usage guidance. |
| `## E2E Strategy` | OPTIONAL | E2E test scope, environment setup, and flakiness mitigation. |
| `## CI Pipeline` | OPTIONAL | CI workflow description, parallelization, and caching. |
| `## Coverage` | OPTIONAL | Coverage measurement, enforcement, and exclusion rationale. |
| `## Test Data & Fixtures` | OPTIONAL | Test data management, factories, seeding, and cleanup. |
| `## Linting & Static Analysis` | OPTIONAL | Linting rules, formatter config, and type checking. |

## Cross-References

- SHOULD reference `@ARCHITECTURE.md` for layer boundaries that determine test scope.
- SHOULD reference `@SECURITY.md` for security-specific test requirements.
- MAY reference `@API.md` for API contract testing scope.
- MAY reference `@SCHEMA.md` for database test fixtures and migration testing.

## Validation Rules

1. Every `Framework` entry MUST include a `type` field.
2. Every `Framework` entry MUST include a `package` field.
3. `pyramid` values SHOULD sum to 100 (tolerance: +/- 5).
4. `coverage.target` MUST be between 0 and 100 inclusive.
5. `ci.workflows` MUST contain at least 1 entry when `ci` is present.
6. `frameworks` MUST contain at least 1 entry.

## Conflict Detection

- `ci.provider` SHOULD match the CI configuration files present in the repository.
- Framework `package` values SHOULD match entries in `package.json`, `requirements.txt`, or equivalent dependency files.
- If `coverage.enforced` is `true`, the CI workflow SHOULD include a coverage gate step.
