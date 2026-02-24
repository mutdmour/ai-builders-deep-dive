# OpenHands — Deep Dive: How It Works Under the Hood

## Overview

OpenHands (formerly OpenDevin, by All Hands AI) is an **open-source, self-hostable
AI software agent platform**. Unlike the cloud builders (Lovable, Bolt, v0) or the
IDE tools (Cursor), OpenHands is closest to an autonomous agent framework — it spins
up a full sandboxed Linux environment and lets an AI operate inside it like a human
developer would: editing files, running terminals, browsing the web, running tests,
and iterating based on results.

It's MIT-licensed, backed by academic research (ICLR 2025 paper), and supports any
LLM — Claude, GPT-4o, Gemini, or open-source models like Qwen and Devstral.

Key claim: **57.4% on SWE-Bench Verified** (solving real GitHub issues) as of their
2024 paper — the highest benchmark score for an open-source agent at the time.

---

## Core Abstraction: The Event Stream

Everything in OpenHands is modeled as an **immutable, append-only event log**. This
is the single source of truth for the entire agent session.

```
EventLog (append-only)
  [0]  UserMessageEvent      "Fix the login bug"
  [1]  ActionEvent           execute_bash("pytest tests/auth/")
  [2]  ObservationEvent      "FAILED tests/auth/test_login.py::test_jwt_expiry"
  [3]  ActionEvent           str_replace_editor(file="auth.py", old="...", new="...")
  [4]  ObservationEvent      "File updated"
  [5]  ActionEvent           execute_bash("pytest tests/auth/")
  [6]  ObservationEvent      "PASSED 14/14"
  ...
```

**Why event sourcing?**
- **Deterministic replay** — crash mid-task? Replay from event 0 to restore exact state
- **Incremental persistence** — each event serialized to a JSON file; only new events
  written on resume
- **Pause/resume** — agent can be stopped between any two events, state fully recovered
- **Auditability** — full trace of every action and result, useful for debugging and
  research evals

The agent's step function is simple: receive the current event history → produce the
next action event. It never holds mutable state internally.

---

## The CodeAct Framework

OpenHands' default agent is **CodeActAgent**, based on the CodeAct paper. The core
insight: instead of defining a fixed set of tools the LLM calls via function-calling
APIs, let the LLM write **actual executable code** as its actions.

### Function-calling vs CodeAct

| Aspect | Traditional Function-Calling | CodeAct |
|---|---|---|
| Action representation | Abstract JSON function signatures | Executable code |
| Flexibility | Limited to predefined tool set | Any code the LLM can write |
| Composability | One tool call at a time | Multi-step code blocks |
| State persistence | External tracking | Variables persist across turns in IPython |
| Learning | Requires tool API knowledge | LLM codes naturally |

### What "Code as Action" Looks Like

Rather than emitting:
```json
{"tool": "run_bash", "args": {"command": "pytest tests/"}}
```

The agent emits:
```python
# CodeAct action
result = execute_bash("cd /workspace && pytest tests/auth/ -v 2>&1")
print(result)
```

The Python/bash code runs in the sandbox, and stdout/stderr become the next
observation event fed back to the LLM.

This means the agent can write complex multi-step programs as a single action —
install dependencies, modify files, run tests, parse output, all in one block —
rather than making separate tool calls for each step.

---

## The Sandbox: Docker Runtime

Every session runs inside a **dedicated Docker container**, not on the host machine.
This is non-negotiable for safety — the agent can run arbitrary shell commands.

### Container Contents

Each container bundles:
- Full Linux environment (bash, standard utilities)
- **REST API server** — receives action events, executes them, returns observations
- **Jupyter IPython server** — persistent Python kernel for stateful code execution
- **Chromium browser** (Playwright-based) — for web browsing actions
- **VS Code Web** — optional, exposes browser-based editor
- **VNC desktop** — optional, full GUI access for desktop automation tasks
- The workspace files (mounted or cloned)

### How Actions Flow Through the Runtime

```
Agent (Python process, outside container)
  ↓ POST /execute  (action JSON)
REST API server (inside container)
  ↓
  ├── CmdRunAction       → bash shell (persistent, stateful)
  ├── IPythonRunCell     → Jupyter kernel (persistent, variables survive turns)
  ├── BrowserInteract    → Playwright Chromium
  └── FileEdit           → str_replace_editor or LLM-based edit_file
  ↓ observation JSON
Agent receives result, appends to event log, generates next action
```

The bash shell and IPython kernel are **persistent across turns** — environment
variables, installed packages, and Python variables all survive between agent steps.
This is critical: `pip install pandas` in turn 3 is available in turn 7.

### Workspace Abstraction (V1 SDK)

The V1 SDK introduced a `BaseWorkspace` abstract class that makes the execution
environment swappable:

| Type | How it works |
|---|---|
| `LocalWorkspace` | Runs directly on host filesystem — no Docker, fast for dev |
| `DockerWorkspace` | Spins up container, delegates over HTTP |
| `APIRemoteWorkspace` | Delegates to cloud-managed runtime (All Hands' hosted service) |

The same agent code works across all three — swap workspace type, same behavior.

---

## Agent Architecture

### Agent Types

| Agent | Purpose |
|---|---|
| `CodeActAgent` | Default generalist — bash, Python, browser, files |
| `BrowsingAgent` | Specialized web browsing with richer DOM interactions |
| Microagents | Community-contributed, keyword-triggered specializations |

### Multi-Agent Delegation

CodeActAgent can spawn sub-agents for tasks outside its core strengths:

```python
# The agent emits this action internally
AgentDelegateAction(
    agent="BrowsingAgent",
    task="Go to the GitHub PR and find the failing test name"
)
```

Sub-agents:
- Run as independent conversations
- Inherit the parent's model configuration and workspace context
- Their events are nested within the parent's event log
- Parent resumes after sub-agent completes and returns a result

This is OpenHands' answer to tool specialization — rather than one generalist with
a big tool list, delegate to a focused agent that knows one domain well.

### Microagents

Microagents are **prompt snippets stored as markdown files** in a registry. They
activate automatically when keywords appear in the user's message or agent's output:

- Triggered by keyword matching (e.g., "npm" → injects Node.js best practices context)
- Can be project-specific (`.openhands/microagents/` in your repo) or global
- Community-contributed specializations for GitHub, npm, Docker, etc.
- No code change required — just a markdown file with a `triggers:` frontmatter

This is how OpenHands gets framework/tool-specific knowledge without bloating the
base system prompt.

---

## V1 SDK: Four-Package Architecture

The V1 rewrite (2024) split the original monolith into four clean packages:

```
openhands.sdk         Core abstractions: Agent, Conversation, LLM, Tool, MCP
openhands.tools       Concrete tool implementations (bash, IPython, browser, files)
openhands.workspace   Execution environments (Local, Docker, RemoteAPI)
openhands.agent_server  REST/WebSocket API server
```

### Stateless Agents

Agents in V1 are **immutable specifications** — no mutable state inside the agent
object itself:
- LLM config, tool list, security policy — all frozen at construction
- All mutable context lives in `ConversationState` (the event log + metadata)
- Agents serialize to JSON and can be transmitted across process boundaries

This makes agents trivially cacheable, testable, and distributable.

### Event Hierarchy

Not all events are created equal — only some are shown to the LLM:

```
LLMConvertibleEvent (sent to LLM)
  ├── MessageEvent          User/agent text
  ├── ActionEvent           What the agent did
  ├── ObservationBaseEvent  What happened as a result
  ├── SystemPromptEvent     System instructions
  └── CondensationSummaryEvent  Compressed history summary

Internal Events (bookkeeping only, not in LLM context)
  ├── ConversationStateUpdateEvent
  ├── CondensationRequest
  └── PauseEvent
```

---

## Context Condensation

Long agent sessions accumulate thousands of events. Without management, costs scale
quadratically — each new turn sends the full history to the LLM.

OpenHands uses `LLMSummarizingCondenser`:

### How It Works

1. **Trigger**: When event history exceeds a token threshold
2. **Summarize**: An LLM call summarizes the oldest N events into a
   `CondensationSummaryEvent`, focusing on:
   - User's original goals
   - Progress made so far
   - Key technical details (critical files, failing tests, discovered bugs)
   - What still needs to be done
3. **Drop**: The summarized events are removed from the active context
4. **Replace**: The summary event takes their place

```
Before condensation (20 events, ~40k tokens):
  [0..19]  Full event log

After condensation:
  [summary]  "User wants to fix auth bug. Installed pytest, found issue in
              jwt_decode() in auth/utils.py line 47. Tests passing except
              test_refresh_token which still fails."
  [15..19]  Most recent 5 events (kept verbatim)
```

### Cost Impact

- Baseline (no condensation): cost per turn scales **quadratically** over session length
- With condensation: scales **linearly** — eventually stabilizes at ~50% of baseline cost
- Cache efficiency: condensation only triggers at threshold, maximizing prompt cache
  hits between triggers

### Custom Condensers

Developers can extend `RollingCondenser` or `CondenserBase` to implement custom
strategies — e.g., always keep test output events verbatim, summarize only
exploratory browsing events.

---

## Security Model

### LLMSecurityAnalyzer

Before any action executes, an optional security layer rates it:

```
Action proposed by agent
  ↓
LLMSecurityAnalyzer
  └── Rates action: LOW / MEDIUM / HIGH risk
  ↓
ConfirmationPolicy
  └── LOW    → auto-approve
  └── MEDIUM → auto-approve or ask (configurable)
  └── HIGH   → require explicit user confirmation
```

Example HIGH-risk actions: `rm -rf`, network requests to unknown hosts,
modifying SSH keys, writing to `/etc/`.

### ConfirmationPolicy

Three modes:
- `CONFIRM_ALL` — user approves every action (safest, slowest)
- `AUTO_APPROVE_ALLOW_LIST` — auto-approve known-safe tools only
- `AUTO_APPROVE_ALL` — fully autonomous (fastest, highest risk)

The policy can be applied differently per tool — e.g., auto-approve file reads,
require confirmation for bash commands, block network access entirely.

---

## Secret Management

OpenHands has a first-class `SecretRegistry` built into the V1 SDK — the only tool
in this comparison to treat secrets as a core architectural concern rather than an
afterthought.

### How It Works

**Static secrets:**
```python
conversation.update_secrets({"STRIPE_API_KEY": "sk-live-..."})
```

**Dynamic secrets** (from external credential stores):
```python
class VaultSecretSource(SecretSource):
    def get_value(self) -> str:
        return vault_client.get_secret("stripe_key")

conversation.update_secrets({"STRIPE_API_KEY": VaultSecretSource()})
```

### Injection and Masking

1. When the agent emits a bash command referencing `$STRIPE_API_KEY`, the registry
   detects the reference and injects it as an environment variable into the
   container at execution time
2. The actual value is **never in the event log** — only the variable name appears
3. If a secret value appears in command output (e.g., an `echo` accidentally prints
   it), the registry **masks it** in the observation before it's written to the
   event log or shown in the UI

**Key guarantee:** secret values never appear in LLM context. The LLM sees
`$STRIPE_API_KEY` as a name, not as `sk-live-abc123...`.

### .env File Handling

OpenHands can detect and import `.env` files via the secrets UI:
- Prompts user to import detected `.env` files securely
- Imported secrets go into the SecretRegistry, not raw into the event log
- Ensures imported secrets are masked even if the agent reads the file

---

## Architecture Summary Diagram

```
[User / Web UI / API]
        |
        | UserMessageEvent
        ↓
[ConversationState]
  EventLog (append-only, persisted as JSON files)
  Metadata (agent_status, cost stats, delegation tracking)
        |
        ↓
[CodeActAgent (stateless)]
  Reads event history
  Builds LLM message sequence
        |
        | LLM API call
        ↓
[LLM — Claude / GPT-4o / Gemini / Qwen / ...]
  Generates next action as code
        |
        ↓
[LLMSecurityAnalyzer]
  Rates risk → ConfirmationPolicy → approve / block / ask
        |
        ↓
[Workspace — Docker container]
  ├── bash shell (persistent)
  ├── IPython kernel (persistent, stateful)
  ├── Playwright Chromium
  └── File system
        |
        | ObservationEvent (stdout/stderr/DOM/screenshot)
        ↓
[SecretRegistry — masks values in output]
        |
        ↓
[ConversationState — event appended]
        |
        ↓
[LLMSummarizingCondenser — if threshold hit]
  → Summarize old events → drop → insert summary
        |
        ↓
[Back to CodeActAgent — next step]
```

---

## OpenHands vs Others

| Dimension | OpenHands | Cursor | Lovable | Bolt.new | v0 |
|---|---|---|---|---|---|
| Type | Autonomous agent framework | Local IDE (VS Code fork) | Cloud builder | Browser IDE | Cloud builder |
| Open source | Yes (MIT) | No | No | Yes (frontend) | No |
| Execution env | Docker sandbox (any machine) | Local machine | Lovable cloud | WebContainers (browser) | Vercel cloud |
| Supported languages | Any (Python, JS, Ruby, Go, …) | Any | React + Vite only | Any npm | Next.js only |
| Agent autonomy | High — runs unsupervised loops | Medium — proposes diffs | Low — one prompt → one change | Low | Low |
| File editing | CodeAct (code execution) | Fast Apply model | Search-replace | Full file rewrite | MDX blocks |
| Browser | Yes (Playwright, built-in) | Yes (headless Chromium) | No | No | No |
| Error feedback | Bash/IPython output (live) | Shadow workspace (LSP) | None | None | AutoFix model |
| Multi-agent | Yes (delegation, microagents) | No | No | No | No |
| Context management | LLMSummarizingCondenser | Windowing + compaction | Unknown | Unknown | Prompt caching |
| Secret management | SecretRegistry (masking, injection) | None built-in | Supabase Vault | Built-in secrets UI | Vercel env vars |
| Deployment | None (BYO) | None (BYO) | Lovable Cloud CDN | Netlify | Vercel (native) |
| LLM | Any (open or proprietary) | Multi-model | Gemini 2.5 Flash | Claude 3.5 Sonnet | Claude Sonnet + AutoFix |
| Self-hostable | Yes | N/A (local IDE) | No | No | No |

---

## What Makes OpenHands Architecturally Distinct

1. **Event sourcing as the core** — not a bolt-on. Every other system uses
   conversation history as a fuzzy approximation of state. OpenHands' event log
   enables true replay, recovery, and auditability.

2. **CodeAct over function-calling** — the agent writes real code rather than
   selecting from a fixed tool menu. This makes it more flexible but also
   less predictable — the action space is unbounded.

3. **Docker-first isolation** — the sandbox isn't optional or browser-based. It's
   a real container, so the agent can run databases, servers, compilers, or anything
   else that installs via apt/pip/npm. Lovable/Bolt/v0 can't do this.

4. **SecretRegistry as first-class** — the only tool here with masking built into
   the observation pipeline. Values never reach LLM context.

5. **Research-grade** — the codebase is designed for both production use and academic
   evaluation. SWE-Bench scores are computed against the same OpenHands codebase,
   not a separate eval harness.

---

## Sources

- [OpenHands Paper (ICLR 2025)](https://arxiv.org/abs/2407.16741)
- [OpenHands V1 SDK Paper](https://arxiv.org/html/2511.03690v1)
- [OpenHands GitHub Repository](https://github.com/OpenHands/OpenHands)
- [CodeAct Agent README](https://github.com/OpenHands/OpenHands/blob/main/openhands/agenthub/codeact_agent/README.md)
- [Secret Registry Docs](https://docs.openhands.dev/sdk/guides/secrets)
- [Context Condenser Docs](https://docs.openhands.dev/sdk/guides/context-condenser)
- [OpenHands Context Condensation Blog](https://openhands.dev/blog/openhands-context-condensensation-for-more-efficient-ai-agents)
