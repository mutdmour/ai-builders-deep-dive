# Codex Web — Deep Dive: How It Works Under the Hood

## Overview

Codex (2025) is OpenAI's **cloud-based software engineering agent** — distinct from
the original Codex model (2021, the GPT-3-based code completion model that powered
early Copilot). The 2025 product is a full autonomous coding agent: you give it a
task, it spins up a cloud sandbox, clones your repo, writes code, runs tests, and
opens a pull request.

It lives in ChatGPT (Plus/Pro/Team/Enterprise) and as a standalone web app. It's
positioned as an **asynchronous background agent** — you submit a task and come back
when it's done, rather than an interactive chat pair-programmer like Cursor.

Key stat: **codex-1 scores 75% on SWE-Bench Verified**, exceeding o3 high-effort
(70%) — the benchmark for solving real GitHub issues autonomously.

---

## The Model: codex-1

Codex Web is powered by **codex-1**, a version of OpenAI o3 fine-tuned specifically
for software engineering.

### How it differs from o3

| Dimension | o3 | codex-1 |
|---|---|---|
| Training | General reasoning via RL | o3 + RL on real-world coding tasks across diverse environments |
| Code style | Functional but generic | Mirrors human PR style and review preferences |
| Instruction adherence | Strong | Stricter — follows coding conventions precisely |
| Test iteration | General | Trained to iteratively run tests until passing |
| SWE-Bench Verified | 70% (high effort) | 75% |
| Output | General text + code | Clean patches ready for immediate code review |

The RL training was on **real coding tasks** across varied environments — not just
synthetic benchmarks. The goal was to produce code that a human reviewer would
accept without significant cleanup, not just code that passes tests.

---

## The Sandbox: Cloud Container Per Task

Every task runs in its own **isolated cloud container**. Not shared, not reused
between tasks — each task gets a fresh environment.

### What's Inside the Container

- Full Linux environment
- Pre-installed language runtimes (Python, Node.js, etc.) — versions configurable
- Your repository cloned at the specified branch/commit SHA
- A persistent bash shell for the agent
- All tools the agent needs: test runners, linters, type checkers

### Two Distinct Phases

This is one of Codex's most important architectural details — the container lifetime
is split into two phases with different security postures:

```
Phase 1: SETUP
  ├── Full internet access
  ├── Setup script runs (arbitrary bash)
  ├── Secrets available as env vars
  ├── Install dependencies, pull models, configure tools
  └── Container state snapshot cached

Phase 2: AGENT
  ├── Internet access OFF by default (configurable)
  ├── Secrets REMOVED from environment
  ├── Only env vars configured in environment settings remain
  ├── Agent edits files, runs tests, iterates
  └── Commits changes when done
```

**Why this split?** Setup legitimately needs to reach the internet (npm install,
pip install, pulling configs). But during agent execution, internet access creates
exfiltration risk — the agent could leak code or secrets to arbitrary endpoints.
The default-off policy during the agent phase is a deliberate security design.

### Container Caching (12-hour TTL)

Running a full setup script on every task would be slow. Codex caches the
post-setup container state for up to 12 hours:

```
First task:
  → Clone repo → run setup script → snapshot container state
  → Cache valid for 12 hours

Subsequent tasks (within 12h):
  → Restore snapshot → checkout specified branch
  → (Optional) run maintenance script for branch-specific prep
  → Agent phase begins immediately
```

Cache is automatically invalidated if you change: setup script, maintenance
script, environment variables, or secrets.

---

## The App Server: The Protocol Layer

OpenAI open-sourced the **Codex App Server** (written in Rust) — the protocol layer
that decouples agent logic from client surfaces. The same server binary powers the
CLI, VS Code extension, and web app.

### Why It Exists

Without the App Server, every client (web, CLI, VS Code, mobile) would need to
implement its own agent loop, state management, and streaming logic. The App Server
is a stable API between the agent core and any UI.

### Protocol: Bidirectional JSON-RPC 2.0

All communication uses JSON-RPC 2.0 over two transports:

| Transport | Used by | Notes |
|---|---|---|
| **stdio (JSONL)** | CLI, VS Code extension | Local binary, blocking, simplest |
| **HTTP + SSE** | Web app | Browser → backend HTTP, backend → worker via SSE |
| **WebSocket** (experimental) | Remote/mobile | Bounded queues, overload rejection (-32001) |

Local clients (CLI, VS Code) bundle the Rust binary, launch it as a child process,
and communicate over stdio. The web app is lighter — it sends HTTP requests to a
backend that proxies to a worker container running the App Server inside it.

### Three Core Primitives

Everything in Codex is built around three hierarchical concepts:

```
Thread  (durable session container)
  └── Turn  (one unit of user input + all resulting agent work)
        └── Item  (atomic output unit)
              ├── User message
              ├── Agent message (streaming)
              ├── Reasoning summary
              ├── Command execution (stdout/stderr)
              ├── File diff
              └── Approval request
```

**Thread**: Persistent across reconnects. Can be created, resumed, forked, or
archived. Persists the full event history so any client can reconnect and render
a consistent timeline. Supports forking — start a new thread from any point in
an existing one.

**Turn**: One unit of work initiated by user input. A turn contains all the items
the agent produces in response: reasoning, tool calls, file edits, test output.

**Item lifecycle**: `started` → zero or more `delta` (streaming) → `completed`.
This enables real-time progressive rendering as the agent works.

### Connection Handshake

```
Client → App Server: initialize  (client metadata, optOutNotificationMethods)
App Server → Client: initialized
Client → App Server: thread/create  (or thread/resume)
App Server → Client: stream of notifications (turn/started, item/started,
                      item/delta, item/completed, turn/completed, ...)
```

Clients can opt out of specific notification types (e.g., suppress reasoning
summaries in a minimal UI) via `optOutNotificationMethods` at initialization.

---

## Parallel Execution: Git Worktrees

Codex runs multiple tasks in parallel using **git worktrees** — each task gets its
own checkout of the repository.

```
main working directory (your branch)

worktree-1/  ← task: "fix login bug"        (branch: codex/fix-login)
worktree-2/  ← task: "add dark mode"        (branch: codex/dark-mode)
worktree-3/  ← task: "write API tests"      (branch: codex/api-tests)
```

Git's one-branch-per-worktree rule prevents conflicts — each task operates on an
isolated checkout. All three run simultaneously in separate containers.

When a task completes:
- **Option A**: Agent commits and pushes → PR opened on GitHub automatically
- **Option B**: "Sync with local" — merge changes back to your primary checkout
  (Overwrite or Apply patch modes)

---

## AGENTS.md: Project Instructions

`AGENTS.md` is Codex's mechanism for persistent, project-aware instructions — the
equivalent of a `CLAUDE.md` or system prompt, but hierarchical and version-controlled.

### Discovery Order

Codex reads files in this precedence (highest to lowest):

```
~/.codex/AGENTS.override.md     (global, always wins)
~/.codex/AGENTS.md              (global defaults)
[repo-root]/AGENTS.md           (project-wide)
[subdirectory]/AGENTS.md        (directory-specific, overrides parent)
```

Files are **concatenated from root downward** — deeper files override shallower ones.
Max combined size: 32 KiB (configurable).

### What You Can Control

- Test commands (`pnpm test`, `cargo test`, etc.)
- Lint and type-check commands
- Dependency management preferences
- Security and review policies
- Code style conventions
- Directory-specific rules (e.g., different standards for `packages/api/` vs `packages/ui/`)
- Review guidelines and severity filtering for the GitHub integration

### Fallback Filenames

If `AGENTS.md` doesn't exist, Codex falls back to: `TEAM_GUIDE.md`, `.agents.md`.
Configurable via `~/.codex/config.toml`.

---

## GitHub Integration

Codex plugs directly into GitHub as a bot. Setup installs a GitHub App on your
organization/repo.

### @codex Mentions

**In PR comments:**
```
@codex review                    → Post a code review (P0/P1 issues by default)
@codex fix the CI failures       → Launch a cloud task with PR as context
@codex add tests for this PR     → Launch a task, produce a follow-up commit
```

**Auto-review:** Enable in settings → Codex posts a review on every new PR without
needing an explicit `@codex review` mention.

### Review Behavior

- Reads `AGENTS.md` for review guidelines and focus areas
- Applies guidance from the closest `AGENTS.md` to each changed file (per-directory rules)
- Default severity: P0 (critical) and P1 (major) only — configurable

### Task Output

When a task completes, Codex:
1. Commits changes in the sandbox with a descriptive commit message
2. Opens a PR with the diff for human review
3. Attaches evidence: terminal logs, test output, lint results
4. Links to the full task trace so you can audit every step

---

## Secret Management

Codex has a hard architectural boundary between secrets and the agent phase — the
most conservative design in this series.

### The Core Rule

**Secrets are available only during setup. They are removed before the agent starts.**

| Phase | Secrets | Env Vars | Internet |
|---|---|---|---|
| Setup script | ✅ Available | ✅ Available | ✅ On |
| Agent phase | ❌ Removed | ✅ Available | ❌ Off (default) |

### Why This Distinction?

Secrets (API keys, tokens, passwords) are needed during setup to authenticate
package pulls, configure services, or download artifacts. But once setup is done,
the agent doesn't need them — and keeping them in the agent's environment creates
risk: the agent could accidentally log them, include them in diffs, or (if internet
is enabled) send them to external services.

By removing secrets at the phase boundary, even a compromised or confused agent
can't exfiltrate credentials.

### How Secrets Are Passed

```
1. Add secret in Codex environment settings UI (stored encrypted)
2. Setup script runs: secret available as $MY_SECRET env var
   → Use it: pip install ... --extra-index-url https://user:$MY_SECRET@...
3. Setup completes → container snapshot cached
4. Agent phase begins: $MY_SECRET is gone from the environment
5. Agent operates with only explicitly configured env vars
```

Setup scripts run in a **separate bash session** from the agent — `export VAR=value`
in the setup script does not persist into the agent phase. To share non-sensitive
config values, add them to `~/.bashrc` or configure them explicitly as environment
variables (not secrets) in settings.

### What the Agent Can Still Access

Regular **environment variables** (non-secrets) persist through both phases —
things like `NODE_ENV=production`, `DATABASE_URL` (connection string without
password), feature flags, etc.

---

## Architecture Summary Diagram

```
[User / ChatGPT / Codex Web UI]
        |
        | Submit task
        ↓
[Codex App Server — Rust binary]
  JSON-RPC 2.0 (stdio / HTTP+SSE / WebSocket)
  Thread/Turn/Item event stream
        |
        ↓
[Cloud Container]
  ┌─────────────────────────────────────┐
  │ SETUP PHASE (internet ON)           │
  │  → Clone repo                       │
  │  → Inject secrets as env vars       │
  │  → Run setup script                 │
  │  → Snapshot + cache (12h TTL)       │
  └─────────────────────────────────────┘
        |
        ↓ secrets removed, internet OFF
  ┌─────────────────────────────────────┐
  │ AGENT PHASE                         │
  │  codex-1 (o3 fine-tuned)            │
  │  ┌─── reads AGENTS.md               │
  │  ├─── edits files                   │
  │  ├─── runs terminal commands        │
  │  ├─── runs tests / linters          │
  │  └─── iterates until passing        │
  └─────────────────────────────────────┘
        |
        | (per-task, parallel via git worktrees)
        ↓
[Git commit → GitHub PR]
  + Evidence: terminal logs, test output, diffs
        |
        ↓
[Human review → merge]
```

---

## Codex Web vs Others

| Dimension | Codex Web | OpenHands | Cursor | Lovable |
|---|---|---|---|---|
| Type | Cloud async agent | Self-hosted agent framework | Local IDE | Cloud builder |
| Model | codex-1 (o3 fine-tune) | Any (Claude, GPT, open-source) | Multi-model | Gemini 2.5 Flash |
| Execution env | Managed cloud container | Docker (self-hosted or cloud) | Local machine | Lovable cloud |
| Interaction mode | Asynchronous (submit + return) | Interactive loop | Interactive (inline) | Interactive chat |
| Parallelism | Native (git worktrees) | Via sub-agent delegation | None | None |
| Internet (agent) | Off by default | Configurable | Full | No |
| GitHub integration | Native (@codex, auto-PR) | Manual git operations | None | Bidirectional sync |
| Secrets | Phase-gated (setup only) | SecretRegistry (masking) | None built-in | Supabase Vault |
| Context instructions | AGENTS.md (hierarchical) | Microagents (keyword-triggered) | .cursorrules | System prompt |
| Audit trail | Full task trace + citations | Event log (replay) | Diff only | Git history |
| Open source | Partially (CLI, App Server Rust code) | Yes (MIT) | No | No |
| Self-hostable | No | Yes | N/A | No |

---

## What Makes Codex Web Architecturally Distinct

1. **Two-phase container with secret removal** — the only tool here that
   architecturally enforces secrets can't reach the agent phase. Everything else
   either leaves secrets in scope or relies on developer discipline.

2. **The App Server as a stable API** — publishing a JSON-RPC protocol that works
   identically across CLI, VS Code, and web is genuinely novel. It makes Codex
   embeddable anywhere, and the Rust binary can be shipped as a self-contained
   artifact.

3. **Asynchronous by design** — Codex isn't trying to be a chat interface. It's
   designed for tasks that take 1–30 minutes. Submit, do something else, come back
   to a PR. This is a different mental model than all the other tools.

4. **Native GitHub as the output surface** — the PR is the primary artifact, not
   a diff in a UI. The integration with `@codex` mentions treats GitHub as a
   first-class client, which fits how engineering teams already work.

5. **codex-1 RL training** — being trained on real-world coding tasks across
   diverse environments (not just benchmark data) means the model has internalized
   what "shippable code" looks like, not just "technically correct code."

---

## Sources

- [Introducing Codex — OpenAI](https://openai.com/index/introducing-codex/)
- [Unlocking the Codex Harness: App Server — OpenAI](https://openai.com/index/unlocking-the-codex-harness/)
- [Codex App Server Docs](https://developers.openai.com/codex/app-server/)
- [Codex Cloud Environments Docs](https://developers.openai.com/codex/cloud/environments/)
- [Custom Instructions with AGENTS.md](https://developers.openai.com/codex/guides/agents-md/)
- [Codex GitHub Integration](https://developers.openai.com/codex/integrations/github/)
- [Codex Worktrees](https://developers.openai.com/codex/app/worktrees/)
- [o3 + Codex System Card Addendum — OpenAI](https://openai.com/index/o3-o4-mini-codex-system-card-addendum/)
- [OpenAI Publishes Codex App Server Architecture — InfoQ](https://www.infoq.com/news/2026/02/opanai-codex-app-server/)
