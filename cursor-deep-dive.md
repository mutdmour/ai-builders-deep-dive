# Cursor — Deep Dive: How It Works Under the Hood

## Overview

Cursor is an AI-first IDE built as a **fork of VS Code**. Unlike Lovable, Bolt, or
v0 — which are cloud-hosted builders that generate and deploy apps — Cursor is a
local development tool. You bring your own codebase, your own runtime, your own
deployment. Cursor's job is to make you a faster developer on that codebase.

It reached $500M ARR as of early 2025. The team keeps the core product surface
small but invests deeply in the underlying infrastructure: a custom indexing
pipeline, a fine-tuned apply model, and a novel shadow workspace for lint feedback.

---

## Architecture: Five Core Systems

```
1. Codebase Indexer      — Merkle trees + embeddings → Turbopuffer
2. Context Assembler     — RAG + @-mentions + AST + LSP data
3. LLM Orchestration     — Multi-model routing (frontier + custom fine-tunes)
4. Fast Apply Model      — Speculative edits at 1000 tok/s
5. Shadow Workspace      — Hidden VS Code instance for lint feedback loops
```

---

## 1. Codebase Indexing

When you open a project, Cursor indexes it in the background. This is the
foundation for the `@codebase` context and semantic search.

### Chunking with Tree-sitter

Files are split into semantically meaningful chunks — not arbitrary line ranges.
Cursor uses **tree-sitter** (an incremental AST parser) to split at natural code
boundaries: function definitions, class bodies, import blocks. Each chunk gets
embedded as a unit so the vector represents a complete logical fragment.

### Merkle Tree for Incremental Updates

```
root hash
├── src/ hash
│   ├── auth.ts hash
│   └── db.ts hash
└── lib/ hash
    └── utils.ts hash
```

Cursor computes a Merkle tree of hashes for every file. Every 10 minutes:
1. Local tree is compared against the server's stored tree
2. Only **changed files** (hash mismatches) are re-embedded and re-uploaded
3. Unchanged chunks are reused from AWS cache, keyed by content hash

This makes re-indexing a large codebase after a small change nearly instant.

### Embeddings + Turbopuffer

- Chunks are embedded using Cursor's own embedding model (not OpenAI's)
- Embeddings + metadata (obfuscated file paths, line ranges) stored in
  **Turbopuffer** — a serverless vector database combining vector + full-text search,
  backed by cheap object storage
- **Crucially: raw source code is never stored server-side.** Only embeddings
  and masked metadata. File paths are obfuscated on the client with a secret key
  before transmission.

### Query Time

When you send a chat message, Cursor:
1. Embeds your query
2. Sends to Turbopuffer for nearest-neighbor search → returns metadata (masked paths + line ranges)
3. Local client resolves masked paths → fetches actual code from your disk
4. Assembles retrieved chunks as LLM context

The privacy architecture means Cursor's servers never see your code in plain text,
even at query time.

---

## 2. Context Assembly

The LLM never sees your whole codebase — only a carefully assembled context window.
Cursor builds it from multiple sources:

| Source | How it's included |
|---|---|
| `@File` mention | Full file content injected directly |
| `@Folder` mention | Key files from folder, summarized if too large |
| `@Codebase` | RAG retrieval via Turbopuffer embeddings |
| `@Docs` | External documentation fetched and cached |
| `@Web` | Live web search results |
| Open editor tabs | Recently viewed files get priority |
| Conversation history | Windowed with compaction for large sessions |
| LSP data | Symbol definitions, type info from language server |
| AST analysis | Imported symbols, call graph context |

The context assembler does **hybrid retrieval** — semantic embedding search
combined with regex/ripgrep for exact symbol matches. Symbol-level precision from
the language server is combined with semantic relevance from embeddings.

Token budget is managed with windowing strategies: oldest conversation turns are
summarized/dropped first, preserving recent edits and key file content.

---

## 3. LLM Orchestration

Cursor is model-agnostic at the product layer but opinionated underneath.

### Available Models

Users can pick from: Claude Sonnet/Opus, GPT-4o, Gemini 2.5 Pro, DeepSeek, and
others. The system prompt names Claude 3.5 Sonnet as the primary model.

### Custom Fine-Tuned Models

Beyond frontier models, Cursor ships two proprietary models:

**Copilot++ (Tab autocomplete)**
A custom next-edit prediction model. Unlike Copilot (which completes the current
token position), Copilot++ predicts multi-line edits and cross-file changes — the
"next logical thing you're likely to change," not just the next token. Uses the
codebase index to understand patterns specific to your project.

**Fast Apply (Instant Apply)**
A fine-tuned Llama-3-70B for applying code changes. See Section 4.

### Routing Logic

- **Tab completions** → Copilot++ (custom, local latency optimized)
- **Small edits / narrow changes** → Fast Apply model
- **Agent/Composer tasks** → Frontier model (Claude Sonnet by default)
- **Lint fix iteration** → Fast Apply + Shadow Workspace feedback

---

## 4. Fast Apply: Speculative Edits at 1000 tok/s

This is Cursor's most technically novel component.

### The Problem

Frontier models (GPT-4o, Claude) are good at *planning* code changes but slow and
unreliable at *applying* them to a large existing file. Problems: laziness
(inserting "// ... rest of code"), inaccuracy on large files, high latency.

Cursor's solution: **separate planning from applying.**

```
[Frontier LLM]           [Fast Apply Model]
"Here's what to change"  → "Here's the full modified file"
   (slow, expensive)          (fast, cheap, specialized)
```

### The Fine-Tuned Model

- Base: **Llama-3-70B** (also trained DeepSeek Coder variant)
- Task: given original file + edit instructions → output complete rewritten file
- Training data: ~450 real "fast-apply" prompts + synthetic data from cmd-k prompts
  (80/20 synthetic/real mix)
- Inference: hosted on **Fireworks AI** with custom speculation logic

**Why full-file rewrite, not diffs?**
Three reasons from Cursor's engineering team:
1. More output tokens = more forward passes = better at determining correct solution
2. Models see far more full files than diffs in pretraining — distribution match
3. Tokenizers treat line number sequences (e.g., "123") as single tokens, causing
   models to commit to wrong line numbers early in diff generation

### Speculative Edits Algorithm

Standard speculative decoding uses a smaller draft model to speculatively generate
tokens, then validates them with the large model. Cursor's variant is different:
it uses **deterministic speculation** — no draft model at all.

For code edits, there's a strong prior on what the next tokens will be:
- Unchanged sections of the file are highly predictable
- The algorithm speculatively emits those sections without running the model
- The 70B model only "wakes up" to generate changed sections
- Validated via greedy (temperature=0) generation

**Result:**
- **~13x speedup** over vanilla Llama-3-70B inference
- **~9x speedup** over GPT-4 speculative decoding
- **~1000 tokens/second** (~3500 characters/second) sustained throughput
- Quality: Llama-3-70B-ft nearly matches Claude-3-Opus on edit tasks

---

## 5. Shadow Workspace

To let AI agents see compiler/lint errors *without disrupting your active editor*,
Cursor built the shadow workspace.

### Architecture

A **hidden Electron window** (`show: false`) running a full VS Code instance in
the background, sharing your workspace folder but invisible to you.

```
Your editor (visible)
     ↑ your edits

Shadow workspace (hidden Electron window)
     ├── AI applies proposed change
     ├── Language server analyzes modified code
     ├── Lint/compiler diagnostics returned
     └── AI iterates or accepts
```

### Implementation Details

**IPC between windows**: Electron sandboxing prevents direct renderer-to-renderer
communication. Cursor uses a layered message chain:
```
AI (renderer) → Extension host A → Extension host B (shadow) → Shadow renderer
```
Implemented with **gRPC + Protocol Buffers** (not VS Code's standard JSON
serialization — too slow).

**Language server integration**: Almost all language servers operate on in-memory
file buffers without writing to disk. Cursor creates in-memory `TextModel` copies
in the shadow window, which triggers the LSP to run diagnostics and report back.

**First-request warmup**: A 2-second artificial delay on the first request lets
the language server boot. Cursor writes deliberately broken code to verify the LSP
is running before trusting its output.

### Concurrency Without Multiple Windows

Rather than spawning one shadow window per concurrent AI task (memory expensive),
Cursor serializes multiple AI requests through a single shared window via state
resets:

```
AI-A wants lints for edit A1
  → reset shadow to A1 state → return lints → AI-A gets feedback

AI-B wants lints for edit B1
  → reset shadow to B1 state → return lints → AI-B gets feedback

AI-A wants lints for edit A2
  → reset shadow to A2 state → return lints → ...
```

This works because "AIs can be paused an indefinite amount of time without noticing"
— humans couldn't tolerate this interleaving but AI agents don't care.

**Memory overhead**: The hidden window roughly doubles memory usage. Mitigations:
limited extensions loaded in shadow, 15-minute auto-kill timers, opt-in config.

**Known limitation**: Rust (relies on `cargo check`, requires disk writes) doesn't
work with shadow workspace linting. Most other languages with LSP support do.

---

## 6. Agent Mode & Tool Use

Cursor's agent (Composer) operates autonomously across the codebase using tools.
From the leaked system prompt:

### Available Tools (10 total)

| Tool | Purpose |
|---|---|
| `read_file` | Read a file's contents |
| `edit_file` | Apply changes to a file |
| `create_file` | Create a new file |
| `delete_file` | Delete a file |
| `run_terminal_cmd` | Execute shell commands |
| `search_files` | Ripgrep across codebase |
| `list_dir` | List directory contents |
| `grep_search` | Pattern search |
| `file_search` | Fuzzy filename search |
| `browser` | Headless Chromium for web inspection |

### Key Agent Rules (from system prompt)

- **Explain tool use before calling**: "Tell the user *why* you're using this tool"
- **Never mention tool names** in user-facing responses
- **Read before editing** (except small appends) — prevents editing stale context
- **Group related edits** into single tool calls — avoids partial states
- **Bias toward self-sufficiency**: find answers via tools before asking the user
- **Root cause over symptoms**: fix underlying issues, don't paper over errors
- **Code must run immediately**: include all imports, deps, config files

### Agent Loop

```
User instruction
  → Frontier LLM plans approach
  → Emits tool calls
  → Tools execute (file reads/writes, terminal, search)
  → Shadow workspace checks lints on edits
  → LLM receives tool results + lint feedback
  → Iterates until task complete or blocked
  → Presents diff for user review
```

User sees a git-style diff of all proposed changes before they're applied. Can
accept all, reject all, or accept individual file changes.

---

## What Cursor Doesn't Do

Cursor is a development tool, not a builder/deployment platform:

- **No hosting** — you deploy to whatever you use (Vercel, AWS, etc.)
- **No built-in backend** — you write and run your own server
- **No auto-commits** — changes are proposed as diffs, applied manually
- **No project initialization** — you start from your existing codebase
- **No preview URL** — you run the dev server yourself

This is the fundamental architectural difference from Lovable/Bolt/v0. Those tools
abstract away the entire dev environment. Cursor augments the environment you
already have.

---

## Secret Management

Cursor has **no built-in secrets management**. It's a local IDE — secrets are
wherever you put them, typically `.env` files. This is also where the most
significant security risk lives.

### The Core Problem: Agent Mode Reads .env

When Cursor's agent runs, it can read any file in your project — including `.env`,
`.env.local`, `.env.production`, and similar files. Once read into context:

- The secret value is in the LLM's context window, sent to Anthropic/OpenAI servers
- The model may echo it in generated code (e.g., writing a test file with the
  literal key value hardcoded)
- It may appear in terminal output if the agent runs commands like `echo`

**Documented incidents:**
- A Cursor agent attempted to upload a local file containing an API key without
  explicit user authorization (caught by a third-party tool)
- A developer's Claude Code session included a Gemini API key in a generated test
  file, which was then committed to GitHub

The `.cursorignore` file (equivalent to `.gitignore` for Cursor's context) can
exclude `.env` files — but the agent has been observed bypassing it by using `ls`
and `cat` shell commands when its file read tool was blocked.

### The Right Approaches

**Option 1: .cursorignore (baseline)**
```
# .cursorignore
.env
.env.*
*.pem
*.key
```
Blocks Cursor's file-read tool from accessing these files. Does not block shell
commands run in agent mode — a known gap.

**Option 2: 1Password Hooks Integration (recommended for teams)**

Cursor supports **Hooks** — scripts that execute automatically at specific points
in the agent workflow (configured in `hooks.json` at project, user, or system
level). 1Password built a Hooks Script that:

1. Fires before the agent executes any shell command
2. Validates that required secrets are available via 1Password Environments
3. Injects secrets **into memory only** for the duration of the session
4. Secrets never touch disk or git history

```json
// hooks.json (simplified)
{
  "preToolCall": {
    "shell": "./scripts/inject-secrets.sh"
  }
}
```

The developer authorizes each access request in the 1Password UI. Secrets are
available to the running process but not written anywhere persistent.

**Option 3: Environment-level isolation**
Run Cursor in a containerized environment where:
- `.env` files mount only what's needed for the current task
- Network egress is restricted (agent can't exfiltrate to arbitrary URLs)
- File system access outside the project root is blocked

**Option 4: Separate secret names from values**
Tell the agent about secret *names* only — never paste values into the chat.
The agent writes `process.env.STRIPE_SECRET_KEY` without ever knowing the value.
You populate `.env` manually after generation.

### What Cursor Explicitly Recommends

From the security docs:
- Enable **Dotfile Protection** — prevents AI from modifying `.env`, `.ssh/config`,
  and credential files
- Enable **MCP Tool Protection** — blocks MCP tools from running without explicit
  approval per-call (MCP is the highest-risk exfiltration vector)
- Use **Privacy Mode** — prevents code (and therefore secrets in context) from
  being used in model training

### The Fundamental Tension

Cursor's power comes from deep codebase access. Secrets management is fundamentally
in tension with that — you want the agent to be able to run your app and tests
(which need secrets) but not leak those secrets. There's no perfect solution yet.
The 1Password Hooks approach is the closest to a production-grade answer as of 2025.

---

## Architecture Summary Diagram

```
[Your codebase on disk]
       |
       | tree-sitter chunking
       ↓
[Merkle hash tree] ─── incremental diff ───→ [Cursor servers]
                                                    |
                                              embedding model
                                                    |
                                             [Turbopuffer]
                                           (obfuscated paths
                                            + embeddings only)

[Your query / @-mentions]
       |
       | query embedding → Turbopuffer nearest-neighbor
       | + LSP symbols + AST + open tabs + conversation
       ↓
[Context window assembled locally]
       |
       ↓
[Frontier LLM — Claude Sonnet / GPT-4o / etc.]
  Plans the change
       |
       | "Here's what to change"
       ↓
[Fast Apply Model — Llama-3-70B-ft on Fireworks AI]
  Speculative edits → full rewritten file at 1000 tok/s
       |
       ↓
[Shadow Workspace — hidden Electron window]
  LSP diagnostics → lint errors fed back to LLM
       |
       ↓
[Diff presented to user for review]
       |
       | accept/reject
       ↓
[Changes written to your files]
```

---

## Cursor vs Lovable vs Bolt vs v0

| Dimension | Cursor | Lovable | Bolt.new | v0 |
|---|---|---|---|---|
| Type | Local IDE (VS Code fork) | Cloud builder | Browser IDE | Cloud builder |
| Target user | Professional developers | Non-developers | Rapid prototypers | Designers/devs |
| Codebase | Yours (any language) | Generated only (React) | Generated only (any npm) | Generated only (Next.js) |
| File editing | Fast Apply model (speculative diffs) | Search-replace blocks | Full file rewrites | Full file rewrites |
| Error feedback | Shadow workspace (LSP live) | None | None | AutoFix model (post-gen) |
| Git | Standard git (your own) | Auto-commit per prompt | None built-in | Vercel project per chat |
| Deployment | BYO | Lovable Cloud CDN | Netlify | Vercel (native) |
| LLM | Multi-model + Copilot++ + Fast Apply | Gemini 2.5 Flash | Claude 3.5 Sonnet | Claude Sonnet + AutoFix |
| Context | RAG + LSP + AST + @-mentions | File tree in system prompt | File tree in context | RAG + dynamic docs |
| Open source | No (VS Code base is open) | No | Yes (frontend) | No (Platform API open) |
| Lock-in | None (your files, your git) | Moderate (GitHub sync) | Low (ZIP export) | Low (export/API) |

---

## Sources

- [Cursor Fast Apply / Instant Apply Engineering Blog](https://cursor.com/blog/instant-apply)
- [Shadow Workspace Architecture — Cursor Blog](https://cursor.com/blog/shadow-workspace)
- [How Cursor Actually Indexes Your Codebase — Towards Data Science](https://towardsdatascience.com/how-cursor-actually-indexes-your-codebase/)
- [How Cursor Indexes Codebases Fast — Engineer's Codex](https://read.engineerscodex.com/p/how-cursor-indexes-codebases-fast)
- [How Cursor Works Internally — Aditya Rohilla](https://adityarohilla.com/2025/05/08/how-cursor-works-internally/)
- [Cursor Fast Apply via Speculative Decoding API — Fireworks AI](https://fireworks.ai/blog/cursor)
- [Cursor System Prompt Revealed — Patrick McGuinness](https://patmcguinness.substack.com/p/cursor-system-prompt-revealed)
- [Leaked Cursor System Prompt — jujumilk3](https://github.com/jujumilk3/leaked-system-prompts/blob/main/cursor-ide-sonnet_20241224.md)
- [Cursor Codebase Indexing Docs](https://cursor.com/docs/context/codebase-indexing)
