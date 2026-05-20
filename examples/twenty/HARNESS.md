---
devmd: harness
version: "1.0"
project: Twenty CRM

intelligence:
  sdk: "Vercel AI SDK 6.0"
  providers:
    - name: OpenAI
      models: [gpt-4o, gpt-4o-mini, o3-mini]
      env: [OPENAI_API_KEY]
    - name: Anthropic
      models: [claude-sonnet-4-20250514, claude-haiku-4-20250514]
      env: [ANTHROPIC_API_KEY]
    - name: Google
      models: [gemini-2.5-pro, gemini-2.5-flash]
      env: [GOOGLE_GENERATIVE_AI_API_KEY]
    - name: Azure OpenAI
      models: configurable
      env: [AZURE_OPENAI_API_KEY, AZURE_OPENAI_ENDPOINT, AZURE_OPENAI_DEPLOYMENT]
    - name: AWS Bedrock
      models: [anthropic.claude-sonnet-4-20250514-v1:0]
      env: [AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_REGION]
    - name: Mistral
      models: [mistral-large-latest]
      env: [MISTRAL_API_KEY]
    - name: xAI
      models: [grok-3]
      env: [XAI_API_KEY]
  configuration: per-workspace
  storage: workspace_settings (encrypted at rest)

knowledge:
  sources:
    - type: workspace-data
      access: twenty-orm (permission-scoped)
    - type: record-search
      method: full-text + fuzzy
    - type: activity-timeline
      includes: [notes, emails, tasks, events]

tools:
  definition_api: defineSkill()
  agent_api: defineAgent()
  package: twenty-sdk

guardrails:
  permission: AI_SETTINGS
  token_limits: per-workspace configurable
  model_selection: admin-only

team:
  package: twenty-claude-skills
  purpose: Developer-time AI assistance for Twenty codebase
---

# Twenty CRM — AI Harness

Comprehensive documentation of Twenty's AI integration. Twenty has real, production AI features — not a wrapper or chatbot bolted on.

## Intelligence Layer

### Vercel AI SDK 6.0 Integration

Twenty uses Vercel AI SDK as the AI abstraction layer. This decouples the CRM from any single AI provider.

```typescript
// Server-side AI configuration loading
import { createOpenAI } from '@ai-sdk/openai';
import { createAnthropic } from '@ai-sdk/anthropic';
import { createGoogleGenerativeAI } from '@ai-sdk/google';

function getAIProvider(workspaceSettings: WorkspaceAISettings) {
  switch (workspaceSettings.provider) {
    case 'openai':
      return createOpenAI({
        apiKey: decrypt(workspaceSettings.apiKey),
        baseURL: workspaceSettings.baseUrl ?? undefined,
      });
    case 'anthropic':
      return createAnthropic({
        apiKey: decrypt(workspaceSettings.apiKey),
      });
    case 'google':
      return createGoogleGenerativeAI({
        apiKey: decrypt(workspaceSettings.apiKey),
      });
    // ... azure, bedrock, mistral, xai
  }
}
```

### Provider Configuration (per-workspace)

Each workspace configures its own AI provider. Stored in workspace settings, encrypted at rest. See @SECURITY.md#data-encryption.

```typescript
// Workspace AI settings schema
interface WorkspaceAISettings {
  provider: 'openai' | 'anthropic' | 'google' | 'azure' | 'bedrock' | 'mistral' | 'xai';
  model: string;                    // e.g., "gpt-4o", "claude-sonnet-4-20250514"
  apiKey: string;                   // encrypted
  temperature: number;              // 0.0 - 1.0
  maxTokens: number;                // per-request limit
  baseUrl: string | null;           // custom endpoint (Azure, Bedrock, self-hosted)
}
```

**Configuration UI**: Settings > AI. Requires AI_SETTINGS permission. See @SECURITY.md#rbac, @SCREENS.md#settings-screens.

### Multi-Provider Fallback

Twenty does not implement automatic provider fallback. Each workspace uses exactly one provider. Switching providers is a manual admin action. This simplifies billing, token accounting, and response consistency.

---

## Knowledge Layer

### Workspace Data as Context

AI features have access to workspace data through the same permission-scoped twenty-orm layer as regular API queries. See @ARCHITECTURE.md#twenty-orm.

```typescript
// Skill execution context provides workspace-scoped data access
execute: async (params, context) => {
  const { workspace } = context;

  // Same permission enforcement as GraphQL API
  // User's role determines what data the AI can see
  const people = await workspace.getRepository('person').find({
    where: { company: { name: ILike(`%${params.query}%`) } },
    relations: ['company', 'noteTargets.note'],
    take: 10,
  });

  return people;
}
```

### Data Access Boundaries

| Data Type | AI Access | Constraint |
|---|---|---|
| Records (Person, Company, etc.) | Yes | Scoped by user's object + field + row permissions |
| Notes and task bodies | Yes | Read permission on parent record required |
| Email content | Yes | Read permission on Message + connected account ownership |
| Attachments | Metadata only | File contents not passed to AI (size/cost) |
| Settings and configuration | No | AI cannot read workspace settings |
| Other workspaces | No | Strict tenant isolation. See @SCHEMA.md#multi-tenant |

### Record Search

AI skills use the same search infrastructure as the command menu. See @SCREENS.md#command-menu.

```typescript
// Full-text + fuzzy search available to skills
const results = await workspace.search({
  query: 'Jane CTO Acme',
  objectTypes: ['person', 'company'],
  limit: 10,
});
// Returns: ranked results with match highlights
```

---

## Tools Layer — defineSkill() API

Skills are typed, validated tools that AI agents invoke. See @AGENTS.md#skills.

### Skill Definition

```typescript
import { defineSkill, z } from 'twenty-sdk';

export const enrichCompany = defineSkill({
  name: 'enrich-company',
  description: 'Look up company information and fill in missing fields (employees, revenue, industry)',
  parameters: z.object({
    companyId: z.string().uuid().describe('The ID of the company to enrich'),
    fields: z.array(z.enum(['employees', 'annualRevenue', 'industry', 'description']))
      .optional()
      .describe('Specific fields to enrich. If omitted, enriches all empty fields.'),
  }),
  execute: async ({ companyId, fields }, { workspace, ai }) => {
    // 1. Fetch current company data
    const company = await workspace.getRepository('company').findOne({
      where: { id: companyId },
    });

    if (!company) throw new Error(`Company ${companyId} not found`);

    // 2. Use AI to research and fill gaps
    const enrichment = await ai.generateObject({
      schema: companyEnrichmentSchema,
      prompt: `Research ${company.name} (${company.domainName?.url}) and provide: ${fields?.join(', ') || 'all available info'}`,
    });

    // 3. Update company record
    await workspace.getRepository('company').update(companyId, enrichment);

    return { updated: Object.keys(enrichment), company: { ...company, ...enrichment } };
  },
});
```

### Skill Execution Context

Every skill receives a typed execution context:

```typescript
interface SkillContext {
  workspace: {
    id: string;
    getRepository<T>(objectName: string): WorkspaceRepository<T>;
    search(params: SearchParams): Promise<SearchResult[]>;
  };
  ai: {
    generateText(params: GenerateTextParams): Promise<string>;
    generateObject<T>(params: GenerateObjectParams<T>): Promise<T>;
    streamText(params: StreamTextParams): ReadableStream;
  };
  user: {
    id: string;
    workspaceMemberId: string;
    role: string;
    permissions: string[];
  };
  logger: Logger;
}
```

### Built-in Skills

See @AGENTS.md#built-in-skills for the full list. Key implementation patterns:

| Skill | Input | Output | Side Effects |
|---|---|---|---|
| `search-contacts` | query, filters, limit | Person[] | None (read-only) |
| `get-record-details` | id, objectType | Full record with relations | None (read-only) |
| `create-record` | objectType, data | Created record | DB write + event emission |
| `update-record` | id, objectType, data | Updated record | DB write + event emission |
| `send-email` | to, subject, body, connectedAccountId | Send confirmation | Email sent via IMAP |
| `create-task` | title, body, assigneeId, dueDate, recordId | Created task | DB write + event emission |
| `get-pipeline-summary` | dateRange, groupBy | Aggregated pipeline stats | None (read-only) |

### Skill Registration

Skills from twenty-sdk apps are registered at app installation time:

```typescript
// App manifest (twenty-app.json)
{
  "name": "my-twenty-app",
  "version": "1.0.0",
  "skills": [
    "./src/skills/enrich-company.ts",
    "./src/skills/score-lead.ts"
  ],
  "agents": [
    "./src/agents/research-agent.ts"
  ]
}

// Registration flow:
// 1. User installs app via UI or CLI
// 2. Server loads skill definitions from manifest
// 3. Skills registered in workspace's skill registry
// 4. Skills become available to AI agents in that workspace
```

---

## Tools Layer — defineAgent() API

Agents are autonomous AI entities with instructions, tools, and multi-step reasoning. See @AGENTS.md#agents.

### Agent Definition

```typescript
import { defineAgent } from 'twenty-sdk';
import { searchContacts, searchCompanies, getRecordDetails, createTask, sendEmail } from './skills';

export const salesAssistant = defineAgent({
  name: 'sales-assistant',
  description: 'AI sales assistant that helps manage pipeline and follow-ups',
  instructions: `
    You are a sales assistant for a CRM. Help users:
    1. Find and enrich contact/company information
    2. Track pipeline progress and suggest next steps
    3. Draft follow-up emails and create tasks
    4. Summarize deal history and meeting notes

    Rules:
    - Always cite specific records by name and link
    - When suggesting actions, offer to execute them (create task, send email)
    - Never fabricate data — if you cannot find it, say so
    - Respect the user's permission level — you can only access what they can
  `,
  tools: [searchContacts, searchCompanies, getRecordDetails, createTask, sendEmail],
  model: 'workspace-configured',
  maxSteps: 10,
  timeout: 60000, // 60 seconds max per conversation turn
});
```

### Agent Execution Flow

```
User message (via AI chat panel or API)
  │
  ▼
AgentExecutionService (twenty-server)
  │
  ├── Load agent definition (instructions, tools, model config)
  ├── Load workspace AI settings (provider, model, apiKey)
  ├── Build tool definitions from registered skills
  │
  ├── Call Vercel AI SDK: generateText / streamText
  │   │  Messages: system (instructions) + conversation history + user message
  │   │  Tools: skill definitions as function calling tools
  │   │
  │   ├── LLM decides to call a tool:
  │   │   │  Tool call: search-contacts({ query: "Acme CTO" })
  │   │   │  → Execute skill with workspace context (permission-scoped)
  │   │   │  → Return tool result to LLM
  │   │   └── LLM continues reasoning (up to maxSteps iterations)
  │   │
  │   └── LLM generates final text response
  │
  ├── Stream response to client via SSE
  │
  └── Log execution:
      │  Duration, token usage, tool calls made, model used
      │  See @LOGGING.md
      └── Emit metrics: ai_agent_execution_duration_seconds, ai_token_usage_total
```

### Agent Lifecycle

```
Installation → Registration → Available → Invoked → Executing → Completed/Failed

States:
  REGISTERED   — Agent loaded from app, available for invocation
  EXECUTING    — Currently processing a user message
  COMPLETED    — Response generated successfully
  FAILED       — Error during execution (timeout, provider error, skill error)
  DISABLED     — Admin disabled this agent for the workspace
```

---

## Guardrails

### Permission Control

AI features are gated by the `AI_SETTINGS` permission flag. See @SECURITY.md#rbac.

| Action | Required Permission |
|---|---|
| Configure AI provider | AI_SETTINGS (settings-level) |
| Use AI features (chat, enrich) | No extra permission (inherits object permissions) |
| Install app with skills/agents | DATA_MODEL (modifies workspace capabilities) |
| Invoke agent | Object permissions for data the agent accesses |

### Token Limits

Workspace admins configure token limits to control AI spend:

```typescript
interface AITokenLimits {
  maxTokensPerRequest: number;     // Default: 4096
  maxTokensPerDay: number;         // Default: 100,000
  maxTokensPerMonth: number;       // Default: 1,000,000
  warningThresholdPercent: number; // Default: 80 (alert at 80% usage)
}
```

Token usage tracked per workspace. Approaching limits triggers admin notification. Exceeding limits blocks AI requests until the next period.

### Data Privacy

- AI requests include workspace data as context. Users must understand their data is sent to the configured AI provider.
- API keys are encrypted at rest and never logged. See @SECURITY.md#data-encryption.
- AI provider selection is intentional — no data sent to any provider without explicit workspace configuration.
- Attachment file contents are never sent to AI providers (only metadata: filename, type, size).

### Error Handling

| Error | Behavior | User Impact |
|---|---|---|
| Provider API key invalid | Return 400 with clear message | Admin notified to update settings |
| Provider rate limit (429) | Retry with backoff (max 3) | Slight delay, then response |
| Provider timeout | Return partial or error after 60s | "AI is taking too long. Try again." |
| Skill execution error | Log error, return skill error to LLM | LLM explains the failure gracefully |
| Token limit exceeded | Block request, return 429 | "AI usage limit reached for this period." |

---

## Team — twenty-claude-skills

A separate package (`twenty-claude-skills`) provides Claude Code skills for developer-time assistance with the Twenty codebase. These are distinct from runtime AI features.

### Available Developer Skills

| Skill | Purpose | Output |
|---|---|---|
| `twenty-module` | Scaffold a new domain module | Resolver + service + entity + DTOs + tests |
| `twenty-migration` | Generate a metadata migration | Migration file following Twenty conventions |
| `twenty-component` | Scaffold a twenty-ui component | Component + styles + stories + spec |
| `twenty-hook` | Generate a React hook | Hook + tests following Recoil/Jotai patterns |
| `twenty-e2e` | Generate Playwright E2E test | Page Object Model + test file |

### Developer Skill vs Runtime Skill

| Aspect | Developer Skill (twenty-claude-skills) | Runtime Skill (defineSkill) |
|---|---|---|
| Runs when | During development (Claude Code session) | At runtime (user interaction) |
| Runs where | Developer's machine | Twenty server |
| Access | Full codebase | Workspace data (permission-scoped) |
| Defined by | Twenty core team | Twenty core team + SDK app developers |
| Package | twenty-claude-skills | twenty-sdk |
| AI provider | Claude (via Claude Code) | Workspace-configured provider |

---

## Built-in AI Features

| Feature | UX Trigger | AI Capability Used | Data Flow |
|---|---|---|---|
| Record enrichment | "Enrich" button on Company/Person detail | generateObject → update record | Read record → AI research → write fields |
| Email drafting | Compose email on record detail | streamText with record context | Read record + emails → AI draft → user edits → send |
| Note summarization | "Summarize" button on note/thread | generateText | Read notes/emails → AI summary → display |
| Smart search | Command menu with natural language | generateText + search skill | Parse query → search → rank → display |
| Workflow suggestions | Workflow builder hint | generateObject | Analyze usage patterns → suggest workflow config |

## Metrics

| Metric | Type | Labels |
|---|---|---|
| `ai_request_duration_seconds` | Histogram | provider, model, feature |
| `ai_token_usage_total` | Counter | provider, model, direction (input/output) |
| `ai_request_error_total` | Counter | provider, error_type |
| `ai_skill_execution_total` | Counter | skill_name, status (success/error) |
| `ai_agent_step_total` | Counter | agent_name, step_type (tool_call/text) |

## Cross-References

- @AGENTS.md — Skill and agent definitions, SDK structure, defineObject/defineField
- @SECURITY.md — AI_SETTINGS permission, data encryption, RBAC enforcement
- @RUNTIME.md — Agent execution may trigger async worker jobs
- @API.md — Skills use the same GraphQL/ORM layer as the API
- @LOGGING.md — AI execution tracing and metrics
- @ERRORS.md — AI-related error codes and handling
- @SCREENS.md — AI settings UI, command menu AI mode
- @FLOWS.md — How AI features integrate into end-to-end flows
