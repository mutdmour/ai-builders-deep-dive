# n8n — Deep Dive: How It Works Under the Hood

## Overview

n8n is an open-source workflow automation platform — think Zapier but
self-hostable, code-friendly, and with a visual node-based editor. You connect
services and logic by wiring nodes together on a canvas. It's a TypeScript
monorepo running a Node.js backend, a Vue 3 frontend, and a pluggable node
ecosystem with 400+ integrations.

It runs locally, self-hosted on any server, or on n8n Cloud.

---

## Package Structure

The monorepo uses **pnpm workspaces** with Turbo for build orchestration.

| Package | Name | Role |
|---|---|---|
| `packages/workflow` | `n8n-workflow` | Core types, expression engine, graph traversal |
| `packages/core` | `n8n-core` | Execution engine, node contexts, binary data, encryption |
| `packages/cli` | `n8n` | Express server, REST API, webhook handling, scheduling |
| `packages/@n8n/db` | `@n8n/db` | TypeORM entities, repositories, migrations |
| `packages/frontend/editor-ui` | `n8n-editor-ui` | Vue 3 visual editor |
| `packages/@n8n/api-types` | `@n8n/api-types` | Shared FE/BE TypeScript interfaces |
| `packages/@n8n/di` | `@n8n/di` | IoC dependency injection container |
| `packages/@n8n/config` | `@n8n/config` | Centralized configuration |
| `packages/@n8n/permissions` | `@n8n/permissions` | Authorization logic |
| `packages/nodes-base` | `n8n-nodes-base` | 400+ built-in integration nodes |
| `packages/@n8n/nodes-langchain` | `@n8n/nodes-langchain` | AI/LangChain nodes |
| `packages/@n8n/design-system` | `@n8n/design-system` | Reusable Vue component library |

---

## Workflow Execution Engine

The heart of n8n. Lives in `packages/core`.

### The Core Classes

**`WorkflowExecute`** (`packages/core/src/execution-engine/workflow-execute.ts`)
The orchestrator. Its `run()` method returns a `PCancelable<IRun>` — a
cancellable promise that resolves when execution completes or fails.

**`Workflow`** (`packages/workflow/src/workflow.ts`)
Represents the workflow graph. Holds nodes, connections, settings, and pin data.
Connections are indexed by source node — to find parents you invert the map using
`mapConnectionsByDestination()`.

### Execution Flow

```
WorkflowRunner (CLI layer — handles persistence, hooks, scaling)
  ↓
WorkflowExecute.run(startNode, inputData)
  ↓
  For each node in dependency order:
    ├── Resolve input data from parent node outputs
    ├── Evaluate expressions in parameters ({{ ... }})
    ├── Decrypt & inject credentials
    ├── Build execution context (IExecuteFunctions)
    ├── Call node.execute(context)
    └── Store output → pass to child nodes
  ↓
IRun result (status, data, timing, errors)
```

### Execution Contexts

Each node type gets a purpose-built context object, not a generic interface:

| Context | Used for |
|---|---|
| `ExecuteContext` | Regular node execution |
| `WebhookContext` | Handling incoming HTTP webhook |
| `PollContext` | Polling trigger checks |
| `LoadOptionsContext` | Dynamic dropdown population |
| `HookContext` | Lifecycle hooks |

All contexts implement the same `IExecuteFunctions` interface from the node's
perspective — the difference is in what helpers are available and how data flows in.

### Graph Traversal

`workflow.connections` is indexed by **source node**. This is important:

```typescript
// To find parents of a node — must invert first
const byDest = mapConnectionsByDestination(workflow.connections);
const parents = getParentNodes(byDest, 'NodeName', 'main', 1);

// To find children — direct lookup
const children = getChildNodes(workflow.connections, 'NodeName', 'main', 1);
```

Both utilities are exported from `n8n-workflow` and are the canonical way to
traverse the graph.

---

## The Node System

### How Nodes Are Defined

Every node implements `INodeType`:

```typescript
class MyNode implements INodeType {
  description: INodeTypeDescription = {
    name: 'myNode',
    displayName: 'My Node',
    version: 1,
    properties: [...],   // parameter schema
    credentials: [...],  // required credentials
  };

  async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    // ... process items
    return [items];
  }
}
```

Output is always `INodeExecutionData[][]` — an array of output connections, each
containing an array of items. Most nodes have one output, but nodes like IF have two.

### Node Discovery & Loading

`LoadNodesAndCredentials` service scans packages at startup:

- **`PackageDirectoryLoader`** — loads from installed npm packages
- **`LazyPackageDirectoryLoader`** — defers loading until first use (performance)
- **`CustomDirectoryLoader`** — loads from user-defined custom node directories

The service reads each package's `package.json` for an `n8n` field listing node
and credential file paths. No runtime discovery — all nodes are declared upfront.

### Community Nodes

Installable via npm at runtime from the settings UI. The CLI downloads the package,
validates it, and hot-loads it without a restart (using the dynamic loader).

### Node Versioning

Nodes can have multiple versions. The `description.version` field controls which
version runs for existing workflows. New workflows default to the latest. Old
workflows pin to the version they were saved with — no surprise breakage.

---

## Expression Engine

The `{{ expression }}` syntax that runs inside node parameters.

### Evaluation

Expressions are evaluated by `WorkflowExpression` (`packages/workflow/src/`) using
a sandboxed JavaScript evaluator. They run just before a node executes, after
input data is resolved but before credentials are fetched.

### Data Access via Proxy

Expressions access workflow data through `WorkflowDataProxy`, which controls
exactly what's visible:

| Variable | What it accesses |
|---|---|
| `$input.item.json` | Current item's JSON data |
| `$input.all()` | All items from input |
| `$node.NodeName.json` | Output from a named node |
| `$node.NodeName.context` | Node execution context data |
| `$vars` | Workflow-level variables |
| `$env` | Whitelisted environment variables |
| `$workflow` | Workflow metadata (id, name, active) |
| `$execution` | Execution metadata (id, mode) |
| `$now` / `$today` | Luxon DateTime helpers |
| `$jmespath()` | JMESPath queries |

### Sandboxing

`expression-sandboxing.ts` strips dangerous globals before evaluation:
- Blocks `eval`, `require`, `process`, `global`
- Prevents prototype pollution via safe Object wrapper
- Sanitizes error objects to prevent stack trace leakage
- Restricts access to `window`/`document` in browser context

The expression engine intentionally limits power — it's a transformation layer,
not a general scripting environment. The Code node exists for that.

---

## Trigger & Webhook System

### Webhook Types

**Live Webhooks** — registered in the database when a workflow is activated.
Maps `POST /webhook/{path}` to a workflow. Persists across restarts.

**Test Webhooks** — ephemeral, in-memory only. Created when you click "Test
workflow" in the editor. Expire after one use or when you close the editor.

**Waiting Webhooks** — used by the Wait node. Execution pauses mid-workflow and
registers a webhook that, when called, resumes execution from that point.

### Webhook Request Flow

```
HTTP request → POST /webhook/{path}
  ↓
WebhookHelpers.getWorkflowStartNode()
  ↓ (look up workflow + node from DB)
WebhookHelpers.executeWebhook()
  ↓
WorkflowRunner.runWebhook()
  ↓
WorkflowExecute.run() (starting from webhook node)
  ↓
Response extracted from workflow output
  ↓
HTTP response returned to caller
```

### Polling Triggers

Nodes that poll (e.g., "check RSS feed every 5 minutes") implement `poll()`.
The `ActiveWorkflowManager` schedules these using cron-style intervals, calling
`poll()` at each tick and injecting any new data as if it were a trigger firing.

### Activation Lifecycle

When a workflow is activated:
1. Webhook nodes → registered in `WebhookEntity` table
2. Trigger nodes → `trigger()` called, returns a cleanup function
3. Polling nodes → scheduled in the poll manager

When deactivated, all of the above are reversed.

---

## Scaling: Queue Mode

For production, n8n can run in **queue mode** — a main process handles API/webhooks
and workers handle execution.

### Technology: Bull + Redis

```
Main Process
  ├── Receives trigger / webhook
  ├── Creates execution record in DB
  ├── Pushes job to Bull queue (Redis)
  └── Streams progress to frontend via WebSocket

Worker Process(es)
  ├── Pull jobs from Bull queue
  ├── Run WorkflowExecute
  ├── Write results to DB
  └── Emit progress events (pub/sub over Redis)
```

### Key Classes

- **`ScalingService`** — manages the Bull queue, job creation, and job cancellation
- **`JobProcessor`** — the worker-side job handler; calls `WorkflowExecute`
- **`WorkerServer`** — the worker process entry point; registers job processors

### Job States

`queued → active → completed / failed / delayed`

Jobs include the full execution data payload. Workers are stateless — any worker
can pick up any job.

### Leader Election

In multi-main deployments, one main instance is elected leader. The leader:
- Runs scheduled (cron) workflow triggers
- Monitors and recovers stalled jobs (crashed workers)
- Manages webhook registration consistency

---

## Credential Management

### Storage

Credentials are stored in the `CredentialsEntity` table, encrypted at rest:
- **Algorithm**: AES-256-GCM
- **Key source**: `N8N_ENCRYPTION_KEY` environment variable (required in production)
- The plaintext JSON blob is encrypted; only the key holder can read it

### Access Flow

```
Node calls: const creds = await this.getCredentials('myServiceApi')
  ↓
CredentialsService.getDecryptedCredentials(credentialId, user)
  ↓ (permission check: does this user/project have access?)
Decrypt blob using encryption key
  ↓
Return ICredentialsDecrypted to node
  ↓
Node uses plaintext values (API key, OAuth token, etc.)
```

Credentials never appear in execution logs or node output. They're injected
transiently into the execution context and not stored in `IRun`.

### Credential Types

Defined with `ICredentialType` — similar to node parameters, they describe the
fields a credential has (apiKey, baseUrl, username, password, OAuth tokens, etc.)
and how to test them. The frontend renders the credential form from this schema.

### External Secrets

Enterprise feature. Integrates with:
- AWS Secrets Manager
- Azure Key Vault
- HashiCorp Vault
- GCP Secret Manager

Secrets are fetched at runtime from the external provider rather than stored in
n8n's database. Referenced by name in credentials: `{{ $secret.MY_KEY }}`.

### Credential Sharing & Permissions

`SharedCredentials` entity controls who can use which credential:
- Credentials are owned by a user or project
- Sharing grants read-use access (but not editing or viewing plaintext)
- Enterprise: project-level isolation

---

## REST API & Real-time

### Backend Structure

```
Request → Express Router
  ↓
@Controller decorator (packages/cli/src/controllers/)
  ↓
@Service (packages/cli/src/services/)
  ↓
Repository (packages/@n8n/db/src/repositories/)
  ↓
TypeORM → SQLite / PostgreSQL
```

All services use `@n8n/di` for dependency injection. Controllers are thin — they
validate input, call a service, and return the result.

### Real-time Push

The editor receives live updates while workflows run:

| Transport | Used when |
|---|---|
| **WebSocket** | Default; full duplex, lower latency |
| **SSE** | Fallback when WebSocket isn't available |

Events pushed to the frontend:
- Execution started/finished
- Node started/finished (per-node progress)
- Execution log entries
- Workflow activated/deactivated by another session

The Push system in `packages/cli/src/push/` abstracts over both transports — the
rest of the codebase just calls `push.send(sessionId, eventType, data)`.

---

## Database & Entities

**ORM**: TypeORM. Supports **SQLite** (default, development) and **PostgreSQL** /
**MySQL** (production).

### Key Entities

| Entity | Purpose |
|---|---|
| `WorkflowEntity` | Workflow definition (nodes JSON, connections JSON, settings) |
| `ExecutionEntity` | Execution record (status, mode, start/end time, workflowId) |
| `ExecutionData` | Input/output data for an execution (separate table, large blobs) |
| `CredentialsEntity` | Encrypted credential blob |
| `UserEntity` | User account |
| `ProjectEntity` | Project (grouping of workflows + credentials) |
| `SharedCredentials` | User↔Credential access mapping |
| `SharedWorkflow` | User↔Workflow access mapping |
| `WebhookEntity` | Registered webhook paths (live webhooks) |
| `AuthIdentity` | OAuth/SAML identity link for SSO |
| `ExecutionAnnotation` | Human annotations on executions |

### Migrations

Schema versioning via TypeORM migrations in `packages/@n8n/db/src/migrations/`.
Run automatically on server start. Separate migration sets per database engine
(SQLite migrations differ from Postgres).

---

## Frontend Architecture

**Stack**: Vue 3 + Vite + Pinia + TypeScript

### State Management (Pinia)

Stores live in `packages/frontend/editor-ui/src/app/stores/`:

| Store | Manages |
|---|---|
| `useNodeTypesStore` | Available node types and their definitions |
| `usePushConnectionStore` | WebSocket/SSE connection state + event dispatching |
| `useWorkflowsStore` | Current workflow, nodes on canvas, connections |
| `useExecutionsStore` | Execution history, current execution progress |
| `useCredentialsStore` | Available credentials for the current user |
| `useUIStore` | Modal state, panel open/closed, global UI state |
| `useHistoryStore` | Undo/redo stack for canvas operations |
| `useLogsStore` | Execution log output |

### Composables

Business logic is extracted into 100+ composables in
`packages/frontend/editor-ui/src/app/composables/`. Components stay thin — they
call composables, composables call stores or APIs.

Examples:
- `useWorkflowExecution` — trigger runs, listen for progress
- `useNodeHelpers` — resolve node display info, validate parameters
- `useCredentialsForm` — render and submit credential forms
- `useCanvasOperations` — add/move/delete nodes on canvas

### Canvas

The workflow canvas uses **@vue-flow/core** (a Vue port of React Flow). Nodes are
rendered as Vue components positioned on an infinite canvas. Connections are SVG
bezier curves.

### Component Architecture

```
Views (pages)
  └── Layouts (page structure)
        └── Components (feature areas: canvas, panel, modals)
              └── Design System components (@n8n/design-system)
```

---

## Architecture Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                   Browser (Vue 3 + Pinia)                     │
│  Canvas (vue-flow) ↔ Stores ↔ Composables ↔ REST API client  │
│                           ↕ WebSocket / SSE (push events)     │
└──────────────────────────────────────────────────────────────┘
                           ↕ HTTP
┌──────────────────────────────────────────────────────────────┐
│                  Express.js Server (CLI)                       │
│  Controllers → Services → Repositories → TypeORM             │
│       ↓                                                        │
│  WorkflowRunner ──────────────────────────────────────────── │
│    ↓ (main mode)              ↓ (queue mode)                  │
│  WorkflowExecute          Bull Queue → Redis                  │
│  (runs in-process)                ↓                           │
│                            Worker Processes                   │
│                           (WorkflowExecute)                   │
└──────────────────────────────────────────────────────────────┘
                           ↕
         ┌─────────────────┼──────────────────┐
         ▼                 ▼                  ▼
    ┌──────────┐    ┌────────────┐    ┌───────────────┐
    │ SQLite / │    │   Redis    │    │ Filesystem /  │
    │ Postgres │    │(queue+pub) │    │ S3 (binaries) │
    └──────────┘    └────────────┘    └───────────────┘
```

---

## Key Architectural Patterns

| Pattern | Where |
|---|---|
| **Dependency injection** | `@n8n/di` — all services, controllers, repositories |
| **Controller → Service → Repository** | All backend API endpoints |
| **Event bus** | Decoupled execution lifecycle events |
| **Plugin system** | Nodes loaded as packages; custom nodes drop-in |
| **Expression sandbox** | Safe JS evaluation via proxy + globals stripping |
| **Encrypted at rest** | Credentials via AES-256-GCM |
| **Cancellable promises** | `PCancelable` for stoppable executions |
| **Shared types** | FE/BE share interfaces via `@n8n/api-types` |

---

## Sources

- [n8n GitHub Repository](https://github.com/n8n-io/n8n)
- `packages/workflow/src/` — expression engine, graph traversal, workflow class
- `packages/core/src/execution-engine/` — WorkflowExecute, node contexts
- `packages/cli/src/` — controllers, services, webhook system, scaling
- `packages/@n8n/db/src/` — entities, repositories, migrations
- `packages/frontend/editor-ui/src/` — Vue stores, composables, canvas
- `/AGENTS.md` — project conventions and development guide
