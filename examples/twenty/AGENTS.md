---
devmd: agents
version: "1.0"
project: Twenty CRM

ai_framework:
  sdk: "Vercel AI SDK 6.0"
  providers:
    - name: OpenAI
      models: [gpt-4o, gpt-4o-mini, o1, o3-mini]
    - name: Anthropic
      models: [claude-sonnet-4-20250514, claude-haiku-3.5]
    - name: Google
      models: [gemini-2.5-pro, gemini-2.5-flash]
    - name: Azure OpenAI
      models: [deployment-configurable]
    - name: AWS Bedrock
      models: [provider-configurable]
    - name: Mistral
      models: [mistral-large, mistral-medium]
    - name: x.ai
      models: [grok-3]
  configuration: >
    AI provider and model configurable per workspace via Settings → AI.
    Requires AI_SETTINGS permission. See @SECURITY.md#rbac.

twenty_sdk:
  package: twenty-sdk
  purpose: "Build apps, skills, and agents that run inside Twenty"
  primitives:
    - name: defineObject
      description: "Define a custom object with fields and relations"
    - name: defineField
      description: "Add a custom field to an existing object"
    - name: defineLogicFunction
      description: "Server-side logic triggered by events or API calls"
    - name: defineSkill
      description: "AI skill (tool) that agents can invoke"
    - name: defineAgent
      description: "Autonomous AI agent with instructions and tools"
  scaffolder: create-twenty-app
  cli: twenty-cli (dev, build, sync)

claude_skills:
  package: twenty-claude-skills
  purpose: "Claude Code integration for developing with Twenty"
  description: >
    Pre-built skills that help Claude Code understand and work with the
    Twenty codebase. Loaded via Claude Code's skill system.
---

# Twenty CRM — AI & Agent Architecture

## AI Integration Overview

Twenty integrates AI at multiple levels:

```
Level 1: Built-in AI Features
  └── Powered by Vercel AI SDK 6.0
  └── Multi-provider (OpenAI, Anthropic, Google, Azure, Bedrock, Mistral, x.ai)
  └── Configurable per workspace

Level 2: Skills (defineSkill)
  └── Tools that AI can invoke
  └── Access CRM data via twenty-orm
  └── Defined by SDK apps or built-in

Level 3: Agents (defineAgent)
  └── Autonomous AI with instructions + tools
  └── Can invoke multiple skills in sequence
  └── Defined by SDK apps or built-in

Level 4: Claude Code Skills (twenty-claude-skills)
  └── Developer-time AI assistance
  └── Understands Twenty codebase conventions
```

## Vercel AI SDK Integration

### Provider Configuration

```typescript
// Workspace AI settings (stored in workspace settings)
{
  "ai": {
    "provider": "openai",          // or anthropic, google, azure, bedrock, mistral, xai
    "model": "gpt-4o",
    "apiKey": "sk-...",            // encrypted at rest
    "temperature": 0.7,
    "maxTokens": 4096,
    "baseUrl": null                // custom endpoint for Azure/Bedrock
  }
}
```

### Built-in AI Features

| Feature | Description | Trigger |
|---|---|---|
| Record enrichment | Fill in missing company/person data from web | Manual or on creation |
| Email drafting | Draft email replies with CRM context | Email compose view |
| Note summarization | Summarize long notes and email threads | Note detail view |
| Smart search | Natural language search across all objects | Command menu |
| Workflow suggestions | Suggest automations based on usage patterns | Workflow builder |

## Skills (defineSkill)

Skills are tools that AI agents can invoke. Each skill has a name, description, input schema, and execution function.

```typescript
// Example: Search contacts skill
import { defineSkill } from 'twenty-sdk';

export const searchContacts = defineSkill({
  name: 'search-contacts',
  description: 'Search for contacts (people) in the CRM by name, email, company, or job title',
  parameters: {
    type: 'object',
    properties: {
      query: { type: 'string', description: 'Search query' },
      filters: {
        type: 'object',
        properties: {
          company: { type: 'string' },
          jobTitle: { type: 'string' },
          city: { type: 'string' },
        },
      },
      limit: { type: 'number', default: 10 },
    },
    required: ['query'],
  },
  execute: async ({ query, filters, limit }, { workspace }) => {
    const repository = workspace.getRepository('person');
    const results = await repository.find({
      where: buildSearchFilter(query, filters),
      take: limit,
      relations: ['company'],
    });
    return results.map(formatPersonResult);
  },
});
```

### Built-in Skills

| Skill | Purpose |
|---|---|
| `search-contacts` | Find people by name, email, company |
| `search-companies` | Find companies by name, domain, industry |
| `search-opportunities` | Find deals by name, stage, amount |
| `get-record-details` | Fetch full record with relations and timeline |
| `create-record` | Create a new record of any object type |
| `update-record` | Update fields on an existing record |
| `send-email` | Send email via connected account |
| `create-task` | Create a follow-up task |
| `create-note` | Create a note on a record |
| `get-pipeline-summary` | Summarize pipeline by stage, amount, probability |

## Agents (defineAgent)

Agents are autonomous AI entities with instructions, tools (skills), and memory.

```typescript
// Example: Sales assistant agent
import { defineAgent } from 'twenty-sdk';
import { searchContacts, searchCompanies, getRecordDetails, createTask } from './skills';

export const salesAssistant = defineAgent({
  name: 'sales-assistant',
  description: 'AI sales assistant that helps manage pipeline and follow-ups',
  instructions: `
    You are a sales assistant for a CRM. You help users:
    1. Find and enrich contact/company information
    2. Track pipeline progress and suggest next steps
    3. Draft follow-up emails and create tasks
    4. Summarize deal history and meeting notes

    Always be concise and action-oriented. Reference specific records by name.
    When suggesting actions, create tasks or draft emails directly.
  `,
  tools: [searchContacts, searchCompanies, getRecordDetails, createTask],
  model: 'workspace-configured', // Uses workspace AI settings
  maxSteps: 10,
});
```

### Agent Execution Flow

```
User message → Agent
  │
  ├── Agent reads instructions + conversation history
  ├── Agent decides which tools to use
  ├── Tool call: search-contacts({ query: "Acme CTO" })
  │     └── Returns: [{ name: "Jane Smith", company: "Acme Corp", ... }]
  ├── Tool call: get-record-details({ id: "uuid-jane", type: "person" })
  │     └── Returns: full record with notes, tasks, emails
  ├── Agent synthesizes response
  └── Response to user (+ optional tool calls for actions)
```

## Twenty SDK (twenty-sdk)

### App Structure

```
my-twenty-app/
├── src/
│   ├── objects/
│   │   └── project.ts           # defineObject — custom object
│   ├── fields/
│   │   └── priority.ts          # defineField — custom field
│   ├── logic/
│   │   └── on-deal-won.ts       # defineLogicFunction — event handler
│   ├── skills/
│   │   └── enrich-company.ts    # defineSkill — AI tool
│   ├── agents/
│   │   └── research-agent.ts    # defineAgent — AI agent
│   └── index.ts                 # App entry point
├── twenty-app.json              # App manifest
└── package.json
```

### defineObject Example

```typescript
import { defineObject, FieldType } from 'twenty-sdk';

export const project = defineObject({
  nameSingular: 'project',
  namePlural: 'projects',
  labelSingular: 'Project',
  labelPlural: 'Projects',
  icon: 'IconFolder',
  fields: [
    { name: 'name', type: FieldType.TEXT, label: 'Project Name', isRequired: true },
    { name: 'status', type: FieldType.ENUM, label: 'Status', options: ['Planning', 'Active', 'Completed', 'Archived'] },
    { name: 'budget', type: FieldType.CURRENCY, label: 'Budget' },
    { name: 'startDate', type: FieldType.DATE, label: 'Start Date' },
    { name: 'endDate', type: FieldType.DATE, label: 'End Date' },
  ],
  relations: [
    { to: 'company', type: 'MANY_TO_ONE', label: 'Client' },
    { to: 'person', type: 'MANY_TO_MANY', label: 'Team Members' },
  ],
});
```

### twenty-cli Commands

```bash
npx create-twenty-app my-app     # Scaffold new app
cd my-app
twenty dev                        # Start dev mode (hot reload against local Twenty)
twenty build                      # Build for production
twenty sync                       # Sync objects/fields/skills to Twenty instance
twenty publish                    # Publish to Twenty app marketplace (future)
```

### twenty-client-sdk

Type-safe GraphQL client generation from workspace schema.

```typescript
import { TwentyClient } from 'twenty-client-sdk';

const client = new TwentyClient({
  baseUrl: 'https://my-twenty.com',
  apiKey: 'app_...',
});

// Fully typed — knows about Person, Company, Opportunity, custom objects
const people = await client.person.findMany({
  filter: { company: { name: { eq: 'Acme' } } },
  orderBy: [{ createdAt: 'DescNullsLast' }],
  first: 10,
});

// Type: Person[] with all fields typed
console.log(people[0].name.firstName);
```

## Claude Code Skills (twenty-claude-skills)

Pre-built Claude Code skills for working with the Twenty codebase.

| Skill | Purpose |
|---|---|
| `twenty-module` | Generate a new domain module (resolver + service + entity + tests) |
| `twenty-migration` | Generate a metadata migration |
| `twenty-component` | Generate a twenty-ui component with Storybook story |
| `twenty-hook` | Generate a React hook with tests |
| `twenty-e2e` | Generate a Playwright E2E test with POM |

## Cross-References

- @SECURITY.md#rbac — AI_SETTINGS permission controls provider configuration
- @SCHEMA.md — defineObject creates ObjectMetadata entries
- @API.md — Skills access data through the same API/ORM layer
- @PRODUCT.md — AI as a core product capability
- @RUNTIME.md — Agent execution may trigger async jobs
