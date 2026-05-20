---
devmd: testing
version: "1.0"
project: Twenty CRM

pyramid:
  unit:
    runner: "Jest (backend), Vitest (frontend)"
    coverage_target: "80%"
    scope: "Services, utilities, hooks, state atoms"
  integration:
    runner: Jest
    scope: "Module interactions, database queries, GraphQL resolvers"
    database: "PostgreSQL test schema (per-test transaction rollback)"
  e2e:
    runner: Playwright
    pattern: Page Object Model (POM)
    scope: "Critical user flows across frontend + backend"
    browser: chromium
  visual:
    runner: "Storybook v10.3 + Chromatic"
    scope: "Component visual regression across themes/states"
  lint:
    runner: "Oxlint (custom rules via twenty-oxlint-rules)"
    formatter: Prettier
    type_check: "tsgo (fast TypeScript type checker)"

conventions:
  file_naming: "{module}.spec.ts (unit), {module}.integration-spec.ts (integration), {feature}.e2e-spec.ts (e2e)"
  test_naming: "describe('{ModuleName}') → it('should {expected_behavior} when {condition}')"
  mocking: "jest.mock() for external deps, test factories for data creation"
  isolation: "Each test file runs in isolated context. DB tests use transaction rollback."

ci_checks:
  - name: lint
    tool: Oxlint + Prettier
    blocking: true
  - name: type-check
    tool: tsgo
    blocking: true
  - name: unit-tests
    tool: "Jest + Vitest"
    blocking: true
  - name: integration-tests
    tool: Jest
    blocking: true
    requires: [postgres, redis]
  - name: e2e-tests
    tool: Playwright
    blocking: true
    requires: [full-stack-deployment]
  - name: visual-regression
    tool: Chromatic
    blocking: false
    review: manual
  - name: breaking-change-detection
    tool: "GraphQL schema diff + OpenAPI schema diff"
    blocking: true
  - name: bundle-size
    tool: "Vite bundle analyzer"
    blocking: false
    threshold: "warn if +10% from main"
---

# Twenty CRM Testing

## Test Pyramid

```
          ┌──────────┐
          │   E2E    │  Playwright — critical user flows
          │  (~50)   │  Slowest, most confidence
         ┌┴──────────┴┐
         │  Visual    │  Storybook + Chromatic
         │  (~500)    │  Component appearance
        ┌┴────────────┴┐
        │ Integration  │  Module interactions, DB queries
        │  (~300)      │  Medium speed, good confidence
       ┌┴──────────────┴┐
       │     Unit        │  Functions, hooks, services
       │   (~2000)       │  Fast, focused
       └────────────────┘
```

## Unit Tests

### Backend (Jest)

```typescript
// person.service.spec.ts
describe('PersonService', () => {
  let service: PersonService;
  let repository: MockType<WorkspaceRepository<Person>>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        PersonService,
        { provide: getRepositoryToken(Person), useFactory: mockRepository },
      ],
    }).compile();
    service = module.get(PersonService);
    repository = module.get(getRepositoryToken(Person));
  });

  it('should create a person with composite name fields', async () => {
    const input = { name: { firstName: 'Jane', lastName: 'Smith' } };
    repository.save.mockResolvedValue({ id: 'uuid', ...input });

    const result = await service.create(input, workspaceContext);

    expect(result.name.firstName).toBe('Jane');
    expect(repository.save).toHaveBeenCalledWith(
      expect.objectContaining({
        nameFirstName: 'Jane',
        nameLastName: 'Smith',
      }),
    );
  });
});
```

### Frontend (Vitest)

```typescript
// usePersonFilter.spec.ts
describe('usePersonFilter', () => {
  it('should build GraphQL filter from view filter config', () => {
    const { result } = renderHook(() => usePersonFilter(), {
      wrapper: RecoilRoot,
    });

    act(() => {
      result.current.addFilter({
        field: 'company.name',
        operator: 'eq',
        value: 'Acme',
      });
    });

    expect(result.current.graphqlFilter).toEqual({
      company: { name: { eq: 'Acme' } },
    });
  });
});
```

## Integration Tests

Backend integration tests run against a real PostgreSQL instance with per-test transaction rollback.

```typescript
// person.integration-spec.ts
describe('PersonResolver (integration)', () => {
  let app: INestApplication;
  let dataSource: DataSource;

  beforeAll(async () => {
    const module = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();
    app = module.createNestApplication();
    await app.init();
    dataSource = module.get(DataSource);
  });

  beforeEach(async () => {
    // Start transaction — rolled back in afterEach
    await dataSource.query('BEGIN');
  });

  afterEach(async () => {
    await dataSource.query('ROLLBACK');
  });

  it('should create person and emit event', async () => {
    const response = await request(app.getHttpServer())
      .post('/api')
      .set('Authorization', `Bearer ${testToken}`)
      .send({
        query: `mutation { createOnePerson(data: { name: { firstName: "Test" } }) { id name { firstName } } }`,
      });

    expect(response.body.data.createOnePerson.name.firstName).toBe('Test');
    // Verify event was emitted
    expect(eventBus.emitted('person.created')).toHaveLength(1);
  });
});
```

## E2E Tests (Playwright)

### Page Object Model Pattern

```typescript
// pages/people.page.ts
export class PeoplePage {
  constructor(private page: Page) {}

  async navigate() {
    await this.page.goto('/objects/people');
    await this.page.waitForSelector('[data-testid="record-table"]');
  }

  async createPerson(firstName: string, lastName: string) {
    await this.page.click('[data-testid="add-button"]');
    await this.page.fill('[data-testid="field-firstName"]', firstName);
    await this.page.fill('[data-testid="field-lastName"]', lastName);
    await this.page.click('[data-testid="save-button"]');
    await this.page.waitForSelector(`text=${firstName} ${lastName}`);
  }

  async filterByCompany(companyName: string) {
    await this.page.click('[data-testid="filter-button"]');
    await this.page.click('text=Company');
    await this.page.fill('[data-testid="filter-value"]', companyName);
    await this.page.click('[data-testid="apply-filter"]');
  }

  async getRowCount(): Promise<number> {
    return this.page.locator('[data-testid="record-row"]').count();
  }
}

// tests/people.e2e-spec.ts
test.describe('People', () => {
  let peoplePage: PeoplePage;

  test.beforeEach(async ({ page }) => {
    await loginAsTestUser(page);
    peoplePage = new PeoplePage(page);
    await peoplePage.navigate();
  });

  test('should create and filter people', async () => {
    await peoplePage.createPerson('Alice', 'Johnson');
    await peoplePage.filterByCompany('Acme');
    const count = await peoplePage.getRowCount();
    expect(count).toBeGreaterThan(0);
  });
});
```

### E2E Test Suites

| Suite | Coverage |
|---|---|
| Auth | Login, signup, SSO, 2FA, password reset |
| People | CRUD, filtering, sorting, inline edit, bulk actions |
| Companies | CRUD, relation to people, domain enrichment |
| Opportunities | Pipeline kanban, stage transitions, won/lost |
| Views | Create, save, share, filter groups, sort |
| Settings | Workspace, members, roles, data model, integrations |
| Command Menu | Open, search, navigate, create from menu |
| Custom Objects | Create object, add fields, create records |

## Visual Regression (Storybook + Chromatic)

### Storybook Setup

twenty-ui components have stories covering all states:

```typescript
// Button.stories.tsx
export default {
  title: 'Input/Button',
  component: Button,
} satisfies Meta<typeof Button>;

export const Primary: Story = { args: { variant: 'primary', label: 'Click me' } };
export const Secondary: Story = { args: { variant: 'secondary', label: 'Click me' } };
export const Disabled: Story = { args: { variant: 'primary', label: 'Click me', disabled: true } };
export const Loading: Story = { args: { variant: 'primary', label: 'Click me', loading: true } };
export const DarkTheme: Story = {
  args: { variant: 'primary', label: 'Click me' },
  decorators: [withDarkTheme],
};
```

### Chromatic CI

- Runs on every PR targeting `main`
- Captures screenshots of all stories in light + dark themes
- Diffs against baseline screenshots
- Visual changes require manual approval before merge
- Automatically accepts unchanged components

## Breaking Change Detection

CI workflow runs GraphQL schema diff and OpenAPI schema diff on every PR.

```yaml
# .github/workflows/breaking-change-detection.yml
- name: Check GraphQL breaking changes
  run: |
    # Generate schema from main branch
    npx graphql-inspector diff \
      schema-main.graphql \
      schema-pr.graphql \
      --rule suppressRemovalOfDeprecatedField
    # Exits non-zero if breaking changes detected

- name: Check OpenAPI breaking changes
  run: |
    npx openapi-diff \
      openapi-main.json \
      openapi-pr.json
```

**Breaking changes that block merge:**
- Removed fields or types
- Changed field types (non-nullable → nullable is OK, reverse is breaking)
- Removed enum values
- Changed argument types

**Non-breaking changes (allowed):**
- Added fields, types, enum values
- Added optional arguments
- Deprecated fields (with migration path)

## Custom Oxlint Rules

The `twenty-oxlint-rules` package defines project-specific lint rules.

| Rule | Description |
|---|---|
| `no-hardcoded-colors` | Colors must come from theme tokens. See @DESIGN.md |
| `no-raw-sql` | All queries must go through twenty-orm. See @SECURITY.md |
| `sort-css-properties-alphabetically` | Consistency in Linaria styles |
| `matching-state-variable` | Recoil atoms must match file name |
| `no-state-useRef` | Prefer Recoil/Jotai over useRef for state |
| `max-consts-per-file` | Limit constants per file for readability |
| `no-navigateToRecordPage` | Use dedicated navigation hooks |
| `no-workspaceEntity-decorator-without-gate` | Workspace entities need feature gate |

## Code Quality Tools

| Tool | Purpose | Config |
|---|---|---|
| Oxlint | Linting (fast, Rust-based) | `twenty-oxlint-rules` + `.eslintrc` |
| Prettier | Code formatting | `.prettierrc` |
| tsgo | Fast type checking | `tsconfig.base.json` |
| Vitest | Frontend unit tests | `vitest.config.ts` |
| Jest | Backend unit + integration | `jest.config.ts` |
| Playwright | E2E tests | `playwright.config.ts` |
| Storybook | Component documentation | `.storybook/` |
| Chromatic | Visual regression | CI workflow |

## Cross-References

- @ARCHITECTURE.md — Module structure that tests follow
- @ERRORS.md — Error codes tested in integration tests
- @DESIGN.md — Visual tokens validated by Oxlint rules and Chromatic
- @INFRA.md#ci-cd — CI pipeline that runs all test suites
