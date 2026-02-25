# Windmill — Deep Dive: How It Works Under the Hood

## Overview

Windmill is an **open-source workflow orchestration platform** — a self-hostable alternative to Airflow, Temporal, and n8n for running scripts and workflows as production services. It executes scripts written in Python, TypeScript/JavaScript, Go, Bash, PHP, or SQL on a distributed worker fleet, triggered by schedules, webhooks, events, or manually.

Its architectural thesis is deliberate simplicity: **stateless Rust workers backed by a single PostgreSQL database**. No Kafka, no Redis, no separate message broker. PostgreSQL is the queue, the job store, the audit log, and the secrets vault simultaneously.

Key claim: **26 million tasks per worker per month** at ~100ms average, with cold starts as low as 15ms for TypeScript.

---

## The Core Architecture: Postgres-Backed Job Queue

### Everything Is a PostgreSQL Transaction

The entire execution model is built around a single principle: **every state transition is one atomic PostgreSQL statement**.

```
Jobs table in PostgreSQL

queued   → running  → completed
            ↑
       Worker pulls via
       UPDATE ... SKIP LOCKED
```

Workers use PostgreSQL's `UPDATE ... WHERE status='queued' SKIP LOCKED LIMIT 1` to claim jobs atomically. `SKIP LOCKED` is a PostgreSQL optimization that skips rows locked by other transactions — multiple workers can pull from the same queue without conflicts or coordination overhead.

This eliminates:
- External message brokers (Redis, RabbitMQ, Kafka)
- Distributed consensus protocols
- Split-brain scenarios
- Message acknowledgment complexity

**Job lifecycle:**
1. **Queued**: Row inserted into `queue` table with `scheduled_for` timestamp
2. **Pulled**: Worker atomically claims row (`UPDATE ... SKIP LOCKED`)
3. **Running**: Worker sets status = `running`, executes code
4. **Completed**: Results and logs written; job moved to `completed_job` table

### The API Server + Worker Separation

Windmill runs as two distinct process types:

```
                      ┌─────────────────────┐
Browser/API clients → │  Server (HTTP API)  │  ← handles UI, API, webhook ingestion
                      └─────────────────────┘
                               │
                          PostgreSQL
                               │
                      ┌─────────────────────┐
                      │   Worker processes  │  ← pull jobs, execute code
                      └─────────────────────┘
```

- **Server mode**: Runs the HTTP API + React frontend. Receives triggers, stores jobs in PostgreSQL, serves results.
- **Worker mode**: Stateless job executors. Pull jobs from PostgreSQL, run them in sandboxed subprocesses, write results back.

Workers are fully stateless. Scale horizontally by adding worker replicas — no coordination needed. The database is the shared state.

---

## The Execution Model: Workers and Runtimes

### Worker Execution Loop

Each worker runs one job at a time:

```
1. SELECT ... SKIP LOCKED  (claim a job)
2. Deserialize job parameters
3. Resolve variables/secrets → inject as env vars
4. Spawn subprocess for the target language runtime
5. Execute main() function
6. Capture stdout/stderr as logs
7. Capture return value as JSON
8. Write result + logs to PostgreSQL
9. Loop
```

### Language Runtimes

| Language | Runtime | Cold Start | Notes |
|---|---|---|---|
| TypeScript/JavaScript | Deno | ~15ms | Native sandboxing via permissions |
| Python | Python 3.x | ~60ms | Requires NSJAIL for prod isolation |
| Go | Go runtime | ~20ms | Compiled per job |
| Bash | bash | ~5ms | Minimal overhead |
| PHP | PHP | ~30ms | |
| SQL | Direct DB connection | ~10ms | PostgreSQL, MySQL, BigQuery, etc. |

### The Deno Advantage

TypeScript jobs run on **Deno** rather than Node.js. This is a deliberate architectural choice:

- **Native sandboxing**: Deno's permission model (`--allow-net`, `--allow-env`, etc.) provides application-level isolation without OS-level overhead
- **Single binary**: No `node_modules` to install; imports resolve at runtime via URL or lock file
- **15ms cold start**: Achieved because Deno's V8 isolate startup is faster than Node's initialization path
- **Immutable script versions**: Each script version is content-addressed by hash; the exact same imports always run the same code

### Dedicated Workers: Eliminating Cold Starts Entirely

For high-frequency scripts, Windmill offers **dedicated workers** (Enterprise):

```
Standard worker:               Dedicated worker:
  ┌─ job arrives                ┌─ worker initialized once
  ├─ spawn subprocess (15ms)    ├─ V8 isolate prewarmed
  ├─ execute (varies)           ├─ job arrives → execute immediately
  └─ subprocess exits           └─ next isolate warming in background
```

The dedicated worker keeps a **continuously running V8 isolate** — while one job runs, the next isolate preloads. This pipelines execution and eliminates the ~15ms cold start overhead entirely. Result: ~2ms overhead per job for TypeScript.

**Performance benchmark vs Temporal:**
- Windmill with dedicated workers: 2.092s for 10 × 30s jobs
- Temporal: 2.967s
- Airflow: 116s (Python DAG overhead)

---

## Process Isolation and Sandboxing

Windmill offers three levels of process isolation, from none to full OS-level:

### Level 0: No Isolation (Default in Community Edition)

By default, the Community Edition **disables process isolation at compile time**. Scripts share the worker process's environment — they can read the worker's environment variables and access the filesystem.

This is explicitly documented as a security tradeoff: isolation adds latency overhead, and many internal deployments don't need it.

### Level 1: PID Namespace Isolation

```bash
# In docker-compose.yml
environment:
  - ENABLE_UNSHARE_PID=true
services:
  windmill_worker:
    privileged: true  # Required for PID namespace
```

When enabled, each job runs in a new PID namespace, preventing it from seeing or sending signals to the worker process. This blocks the most common side-channel attacks against the worker.

### Level 2: NSJAIL (Full Sandboxing)

**nsjail** is Google's open-source process isolation tool used in production at Google (e.g., in competitive programming judges). It uses:

- Linux namespaces (PID, mount, network, user, IPC)
- cgroups for resource limits (CPU, memory)
- rlimits for syscall limits
- seccomp-bpf for syscall filtering

```bash
# Enable NSJAIL in worker config
DISABLE_NSJAIL=false
```

With NSJAIL enabled:
- Each job runs in a completely isolated Linux namespace
- No filesystem access to the host
- Network access controllable (can be fully disabled)
- Cannot see other processes
- Memory and CPU limits enforced

NSJAIL takes precedence over PID namespace if both are configured.

**Language-specific requirements:**
- Python: Requires NSJAIL or PID namespace in production (can otherwise read worker secrets)
- Go: Same
- TypeScript (Deno): Deno's native permissions provide application-level sandboxing; NSJAIL adds OS-level on top

### Native Subworkers: High-Throughput Database/API Jobs

For SQL and native REST/GraphQL jobs, Windmill uses **native subworkers** — specialized worker threads optimized for I/O-bound operations:

```
Native worker group
  ├── PostgreSQL subworker (connection pool)
  ├── MySQL subworker
  ├── REST/GraphQL subworker
  └── BigQuery subworker
```

These avoid subprocess spawning overhead entirely and maintain persistent connection pools, making SQL queries execute in ~10ms end-to-end.

---

## Scripts, Flows, and Apps

### Scripts: The Atomic Unit

A Script is a versioned function with a single `main()` entrypoint:

```python
# Python script
def main(spreadsheet_id: str, sheet_name: str = "Sheet1") -> dict:
    # Windmill injects type-annotated parameters as the job input
    data = fetch_sheet(spreadsheet_id, sheet_name)
    return {"rows": len(data), "data": data}
```

```typescript
// TypeScript script
export async function main(spreadsheetId: string, sheetName = "Sheet1") {
    // Import from Windmill hub or npm
    const data = await fetchSheet(spreadsheetId, sheetName);
    return { rows: data.length, data };
}
```

Key properties:
- **Versioned by hash**: Each save creates an immutable version. Running a script at a specific hash is reproducible forever.
- **Auto-generated UI**: Windmill generates a form UI from the `main()` function's typed parameters — no manual UI code needed.
- **Async streaming**: Return an `AsyncGenerator` to stream results incrementally.

### Flows: DAG Orchestration

Flows compose Scripts into directed acyclic graphs:

```
Flow definition (JSON/YAML)
  steps:
    - id: fetch_data
      module: script:u/alice/fetch_spreadsheet
      input_transforms:
        spreadsheet_id: flow_input.spreadsheet_id

    - id: process
      module: script:u/alice/process_rows
      input_transforms:
        data: results.fetch_data.data   # Previous step output

    - id: notify
      module: script:u/alice/send_slack
      branches:
        - condition: results.process.errors > 0
          steps: [...]   # Error branch
```

**Input transforms** are the glue: small JavaScript expressions (evaluated in embedded V8, ~8ms) that map flow inputs, previous step results, variables, and resources into each step's parameters.

For trivial transforms (`results.step1.field`), Windmill compiles them directly to PostgreSQL JSONB queries — no V8 overhead at all.

**Flow features:**
- **Branching**: If/else conditions, multi-branch fan-out
- **Loops**: `for_loop` and `while_loop` step types
- **Approval steps**: Pause flow, wait for human approval via email/Slack
- **Error handling**: Per-step retry, error branches
- **Resume from step**: Re-run a failed flow starting from the failed step
- **Flow state**: Key-value store shared across all steps in one run

### Apps: Low-Code + Full-Code UI

Apps are React-based interfaces that call Scripts as actions:

- **Low-code builder**: Drag components (buttons, tables, forms, charts), connect to Scripts
- **Full-code mode**: Write arbitrary React components
- **Client-side state**: Component state wires directly to Script outputs
- **Context variables**: `ctx.username`, `ctx.workspace`, `ctx.query.param` available in scripts

---

## Worker Groups and Tag Routing

### Tag-Based Job Routing

Every job has a `tag` that determines which worker group processes it. Workers advertise which tags they listen to:

```
Worker group "gpu-workers"   → listens for tags: ["ml", "gpu", "embedding"]
Worker group "fast-workers"  → listens for tags: ["default", "light"]
Worker group "native"        → listens for tags: ["postgresql", "rest", "graphql"]
```

Jobs specify their required tag. A ML training job tagged `gpu` will only run on workers in the GPU group.

### Dynamic Tags

Tags support variable substitution:
- `$workspace` → replaced with the workspace ID at runtime
- `$args[argName]` → replaced with a specific job argument

This enables routing jobs to workspace-specific or parameter-specific worker pools.

### Sizing Formula

```
Workers needed = Peak job arrival rate × Average job duration

Example: 1,000 reports/hour, 30s each:
  = (1000/3600) × 30 = 8.3 → 9 workers minimum

With 2-minute acceptable queue time:
  Workers = (arrival_rate × duration) + (arrival_rate × max_wait)
```

One worker executing 100ms jobs handles ~26 million tasks/month.

---

## Secret Management: Variables and Resources

### Variables and Secrets

All variables are encrypted at rest using workspace-specific symmetric keys. Variables marked as "secret" are write-only from the UI — their values cannot be viewed outside a running script.

**Path-based access control:**
```
u/alice/stripe_key        ← private to user alice
f/payments/stripe_key     ← shared within "payments" folder/group
g/admins/db_password      ← shared with admins group
```

**Injection at execution time:**
```python
# In a script, reference via Windmill's context
import wmill

secret = wmill.get_variable("u/alice/stripe_key")
# Or inject as env var at job creation time
```

The worker resolves variable paths and injects values into the subprocess environment at execution time. The job definition in PostgreSQL never contains the raw secret value — only the variable path.

**Audit trail:** Every `get_variable()` call for a secret generates a `variables.decrypt_secret` audit log event.

### Resources and Resource Types

Resources are JSON objects representing connections to external systems:

```json
{
  "type": "postgresql",
  "data": {
    "host": "db.internal",
    "port": 5432,
    "database": "prod",
    "user": "app",
    "password": "{{u/db/password}}"  ← references a Variable
  }
}
```

Resource types define the JSON schema (database connection shape, API client shape, etc.). The Windmill Hub hosts community-contributed resource types for 100+ services.

Scripts receive Resources as typed parameters — the worker resolves the resource JSON (including Variable references) and passes it to the function.

### LLM Cannot Read Secrets

Unlike Replit (where the AI agent operates in the same environment as secrets), Windmill's AI features are entirely separate from the worker environment. The LLM generates script code; it never executes in the worker environment and cannot access variable values.

---

## AI Features: Windmill AI

Windmill AI integrates LLM-powered code generation into the development experience, not the execution environment.

### Code Generation

Describe the script in natural language → AI generates the `main()` function:

```
"Fetch all rows from a Google Sheet where column B > 100,
 then send a Slack message with a summary"

→ Generates Python/TypeScript using the correct Windmill imports,
  Resource parameter types, and return shape
```

**Models supported:**
- Mistral Codestral (primary for autocomplete — requires Fill-in-Middle endpoint)
- Anthropic Claude 3.5 Sonnet
- OpenAI GPT-4

### AI Error Fix

When a script fails, an "AI Fix" button:
1. Sends the script source + error message + stack trace to the LLM
2. LLM explains the error in plain English
3. LLM suggests a corrected version
4. User reviews and applies

### Flow AI Copilot

In the flow builder:
- Describe the step's goal → AI suggests the right script from the workspace
- Step Input Copilot analyzes previous steps' output schemas → suggests input transform expressions

### Schema-Aware Generation

For SQL scripts, the AI receives the target database's schema. For GraphQL, it receives the API's type definitions. This enables accurate query generation without hallucinating non-existent fields.

---

## Deployment

### Docker Compose (Single Machine)

Windmill ships a three-file setup:

```yaml
# docker-compose.yml (simplified)
services:
  db:
    image: postgres:16
    volumes: [db_data:/var/lib/postgresql/data]

  windmill_server:
    image: ghcr.io/windmill-labs/windmill
    command: ["server"]
    environment:
      - DATABASE_URL=postgres://...

  windmill_worker:
    image: ghcr.io/windmill-labs/windmill
    command: ["worker"]
    environment:
      - DATABASE_URL=postgres://...
    # Scale workers:
    deploy:
      replicas: 4

  caddy:
    image: caddy:2
    # Reverse proxy + automatic TLS
```

Rule of thumb: 1 worker per vCPU, 1–2 GB RAM per worker.

### Kubernetes via Helm

```bash
helm repo add windmill https://windmill-labs.github.io/windmill-helm-charts/
helm install windmill windmill/windmill \
  --namespace windmill \
  --create-namespace \
  --set windmill.baseDomain=windmill.example.com
```

The chart deploys server pods (frontend + API) and worker pods separately. Autoscaling can be configured on the worker deployment independently of the server.

### Cloud-Managed (Windmill Cloud)

Windmill Labs operates a managed cloud version — no self-hosting, same functionality. Enterprise deployments can get dedicated instances.

### Binary Size

Windmill compiles to a single Rust binary (~60MB). The same binary runs in both server mode and worker mode via a `--mode` flag. This makes deployment trivial and container images small.

---

## Open Source and Licensing

| Component | License |
|---|---|
| Core platform (Community Edition) | AGPLv3 (self-hosted) |
| SDK and API clients | Apache 2.0 |
| Enterprise features | Proprietary (EE binary) |

**Enterprise-only features** (proprietary, separate binary):
- Dedicated workers
- Agent workers (HTTP-only, for untrusted networks)
- Advanced RBAC and audit logs
- SAML/SSO
- Git-based deployment workflows
- Distributed work queues

**Open source without restrictions for:**
- Internal organizational use
- API integration into your own product

**Commercial license required for:**
- Re-exposing Windmill UI to your end users
- Embedding and reselling Windmill as part of a product

GitHub: [windmill-labs/windmill](https://github.com/windmill-labs/windmill) (18,000+ stars)

---

## Architecture Summary Diagram

```
[Browser / API / Webhooks / Cron / Events]
        |
        ↓
[Windmill Server — Rust HTTP API + React UI]
        |
        ↓
[PostgreSQL — single source of truth]
  ├── queue table          (pending jobs)
  ├── completed_job table  (results + logs)
  ├── variable table       (encrypted secrets)
  ├── resource table       (connection configs)
  ├── script table         (versioned, hash-addressed)
  └── flow table           (DAG definitions)
        |
        ↓ Workers poll via UPDATE...SKIP LOCKED
[Worker Fleet — Rust worker processes]
  ├── Standard workers (one job at a time)
  │     ├── TypeScript → Deno subprocess (~15ms cold start)
  │     ├── Python    → Python subprocess + NSJAIL (~60ms)
  │     ├── Go        → Go binary (~20ms)
  │     └── Bash/SQL  → direct execution (~5ms)
  │
  ├── Dedicated workers (Enterprise)
  │     └── Prewarmed V8 isolates (~2ms overhead)
  │
  └── Native subworkers
        └── Connection pools for SQL/REST/GraphQL (~10ms)

[AI Layer — separate, not in worker env]
  └── LLM API calls for code generation / error fix
      (never has access to execution environment or secrets)
```

---

## Windmill vs. Others

| Dimension | Windmill | Airflow | Temporal | n8n | Prefect |
|---|---|---|---|---|---|
| Primary language | Rust + Python/TS/Go scripts | Python DAGs | Go | Node.js | Python |
| Queue backend | PostgreSQL | Celery/Redis | Cassandra/MySQL | PostgreSQL | PostgreSQL |
| Cold start (TS) | 15ms | N/A | N/A | N/A | N/A |
| Lightweight task throughput | 26M/worker/month | ~100K/month (Python overhead) | 5M/month | — | — |
| Script languages | 6+ (Python, TS, Go, Bash, PHP, SQL) | Python only | Any (via activities) | JS/Python (limited) | Python |
| Built-in UI builder | Yes (Apps) | No | No | No | No |
| Self-hosted | Yes (AGPLv3) | Yes (Apache 2.0) | Yes (MIT) | Yes (Apache 2.0) | Yes (Apache 2.0) |
| Managed cloud | Yes | Astronomer | Temporal Cloud | n8n Cloud | Prefect Cloud |
| Process isolation | NSJAIL + PID namespaces | None | None | None | None |
| AI code generation | Yes | No | No | Partial | No |
| Secrets | Encrypted Variables + Resources | External (Vault/K8s) | External | Built-in | External |

---

## What Makes Windmill Architecturally Distinct

1. **PostgreSQL as the entire infrastructure** — using `SKIP LOCKED` for a distributed queue means Windmill has zero dependency on Redis, Kafka, or any message broker. One database handles everything. This dramatically reduces operational complexity.

2. **Deno for TypeScript** — running TypeScript in Deno (not Node) gives 15ms cold starts and application-level sandboxing via Deno's permission model without OS overhead. No `node_modules` means no package installation at job time.

3. **Single Rust binary** — server and worker are the same binary, different modes. 60MB container images. No JVM, no Python runtime required to run the platform itself.

4. **Immutable script versioning** — every script version is content-addressed by hash. A job scheduled 6 months ago will run exactly the same code if replayed today.

5. **V8 isolate pipelining** — dedicated workers overlap execution and preloading: while job N runs, job N+1's isolate is warming. This is the same technique V8 uses internally for optimizing JavaScript execution.

6. **AI kept out of execution** — unlike platforms where the AI agent operates inside the execution environment (Replit, Codex), Windmill AI generates code but never executes in the worker environment. It has no access to secrets.

---

## Sources

- [Windmill Docs — Core Concepts](https://www.windmill.dev/docs/core_concepts)
- [Windmill Docs — Security and Process Isolation](https://www.windmill.dev/docs/advanced/security_isolation)
- [Windmill Blog — Fastest Self-Hostable Workflow Engine](https://www.windmill.dev/blog/launch-week-1/fastest-workflow-engine)
- [Windmill Docs — Dedicated Workers](https://www.windmill.dev/docs/core_concepts/dedicated_workers)
- [Windmill Docs — Workers and Worker Groups](https://www.windmill.dev/docs/core_concepts/worker_groups)
- [Windmill Docs — Variables and Secrets](https://www.windmill.dev/docs/core_concepts/variables_and_secrets)
- [Windmill Docs — Scaling](https://www.windmill.dev/docs/advanced/scaling)
- [Windmill Docs — Agent Workers](https://www.windmill.dev/docs/core_concepts/agent_workers)
- [Windmill Benchmark Results](https://www.windmill.dev/docs/misc/benchmarks/competitors/results/conclusion)
- [Deno Blog — Immutable Scripts with Windmill](https://deno.com/blog/immutable-scripts-windmill-production-grade-ops)
- [GitHub — windmill-labs/windmill](https://github.com/windmill-labs/windmill)
