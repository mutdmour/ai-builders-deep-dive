# Bolt.new — Deep Dive: How It Works Under the Hood

## Overview

Bolt.new is StackBlitz's AI-powered full-stack app builder that runs an entire
Node.js development environment **inside your browser tab** — no cloud VM, no SSH,
no Docker. You describe what you want, the AI generates and runs the code, and you
see a live preview immediately.

It went from $0 to $4M ARR in 30 days and $40M ARR shortly after, built by a team
of 15 engineers. The enabling technology is WebContainers — StackBlitz's existing
browser-runtime project that predates Bolt by several years.

The key insight: **the server is your browser tab.**

---

## The Fundamental Architecture: WebContainers

This is what makes Bolt categorically different from Lovable or v0.

### What WebContainers Are

WebContainers is a WebAssembly-based operating system that runs inside a browser
sandbox. It emulates Linux enough to run a full Node.js environment — including npm,
file I/O, processes, and networking — without any remote server involvement.

```
Traditional cloud IDE:
  Browser → WebSocket → Remote VM (Linux) → Node.js process

Bolt.new:
  Browser tab → WebContainer (WASM) → Node.js process (in-browser)
```

### Virtual File System (Rust + WASM)

Bolt built a custom Rust-based filesystem compiled to WebAssembly:

- **POSIX-compliant** — Node.js can't tell it's not a real filesystem
- **SharedArrayBuffer** — a single block of shared memory all Web Workers can
  read/write simultaneously. No JSON serialization, no async message-passing overhead.
- **Atomics API** — provides file locks and atomic writes that Node.js expects
  from a real OS. Prevents race conditions between concurrent processes.
- **Speed** — Rust → WASM runs at near-native speed with no garbage collection pauses

When a Bolt tab opens, the filesystem module is loaded into a SharedArrayBuffer.
All subsequent reads and writes happen directly in shared memory — as fast as a
real disk, but in RAM.

### Process Model

Each Node.js process becomes its own **Web Worker**:

- A Rust "kernel" manages process lifecycle, task queues, and signal handling (SIGTERM, etc.)
- Workers communicate via Atomics (shared memory) rather than postMessage
- A custom TypeScript shell (`jsh`) emulates Bash, dispatching commands to Workers

### Networking

Real networking from inside a browser sandbox requires creative routing:

- **Service Workers** intercept special URLs and route requests to the right Worker
- **WebSocket bridge** maintains hot-reload (HMR) between the Vite dev server
  running in a Worker and the preview iframe
- **TCP relay fallback** — for tools requiring raw sockets, a small relay server
  bridges to the outside world

### npm Install in the Browser

This is one of the most impressive parts. Running `npm install` in a browser normally
takes 30–60 seconds. Bolt gets it to **< 500ms** for popular packages:

- Popular packages are pre-compressed and stored in Bolt's CDN as layered archives
- After the first fetch, they're cached in the browser's HTTP cache
- Custom module loader handles Node.js module resolution (ESM + CommonJS compat)
  by injecting the dependency graph directly rather than resolving it at runtime

### WebContainer Constraints (from System Prompt)

The system prompt explicitly lists what WebContainers **cannot** do:

| Constraint | Detail |
|---|---|
| No native binaries | Only JS/WASM can execute |
| Python | Standard library only — no pip, no third-party packages |
| No C/C++ compiler | No g++ or native build tools |
| No Git | Git is unavailable inside the container |
| Shell | Emulates zsh; specific commands only (cat, node, python3, etc.) |

---

## LLM Integration

### Model

Bolt.new is powered by **Claude 3.5 Sonnet** (Anthropic). StackBlitz chose it
specifically for zero-shot code generation — Claude could understand and execute
complex dev tasks without extensive prompt engineering. This partnership got
StackBlitz from $0 to $4M ARR in 4 weeks.

Users can also select other models (GPT-4o, etc.) in settings, but Claude is the default.

### How the LLM Fits In

```
User prompt
  → Sent to Claude API (edge-hosted for low TTFB)
  → Claude generates boltArtifact XML
  → Bolt's frontend parses the XML
  → Actions dispatched: file writes + shell commands
  → WebContainer executes changes in real time
  → Preview iframe reflects live running app
```

---

## How the AI Edits Code: The Artifact Protocol

### From the Leaked System Prompt

Bolt wraps all code changes in a structured XML format. This is the actual
protocol between the LLM and the runtime:

```xml
<boltArtifact id="project-import" title="Project Files">
  <boltAction type="file" filePath="src/App.tsx">
    // full file contents here
  </boltAction>
  <boltAction type="shell">
    npm install && npm run dev
  </boltAction>
</boltArtifact>
```

### Two Action Types

**`type="file"`** — Write or overwrite a file:
- Path is relative to `/home/project`
- Always outputs the **complete file contents** — no truncation, no placeholders
- Used for both new files and edits to existing files

**`type="shell"`** — Run a terminal command:
- Commands chained with `&&` and run sequentially
- Always includes `--yes` flags for npx to avoid interactive prompts
- **Critical rule**: never re-run a dev server command if one is already running

### Diff vs Full-File for User Edits

When the user edits code manually (not via AI), changes come back to the LLM via
`<bolt_file_modifications>` with two formats depending on size:

- **`<diff>` tag** — GNU unified diff format for concise changes
- **`<file>` tag** — full file content when the diff would be larger than the file

The LLM always applies modifications to the latest version of the file before
generating its response.

### Key Prompt Rules

- **Holistic thinking first** — plan the full change before writing any artifact
- **No monolithic files** — split code across modules
- **Never say "artifact"** — the word is banned from user-facing responses
- **No verbose explanations** unless explicitly asked

---

## Deployment

### No Built-in Git

Bolt does not sync to GitHub automatically (unlike Lovable). There's no built-in
version history or commit-per-prompt model. You download the project as a zip or
connect it to GitHub manually.

### Deployment = Netlify

Bolt's native deployment target is **Netlify**:

1. Click Deploy → Bolt builds the project (Vite/Next.js/etc.)
2. Bolt pushes the build output to a Netlify site it controls
3. You get a random `*.netlify.app` URL immediately (~1 min)
4. "Claim URL" transfers the Netlify site into your own Netlify account

```
[WebContainer build]
  → Vite/Next.js build runs in browser
  → Build output zipped
  → Pushed to Netlify API
  → *.netlify.app URL live
  → (Optional) Transfer to your Netlify account
```

### Secret Management

Bolt has a built-in Secrets UI for server-side credentials:

**How it works:**
1. Open the project dashboard → database icon → "Secrets" in left sidebar
2. Enter a name and value → click "Create secret"
3. Claude Agent automatically prompts you to add a secret when it detects one is
   needed (e.g., adding an OpenAI integration triggers a prompt for your API key)
4. Secrets are injected into server functions at runtime as environment variables
5. They are **server-side only** — never accessible in the client browser

**What is and isn't protected:**

The built-in secrets UI handles intentional secrets. However, Bolt also generates
code that runs entirely in the browser (WebContainers), which creates a structural
risk: anything in frontend JavaScript is visible to the browser. Bolt's guidance
is clear — API keys must live in server functions, never in client-side code.

**The .env problem:**
For local development or projects exported to GitHub, secrets are typically stored
in a `.env` file. Bolt generates code referencing `process.env.MY_KEY` but does
not manage `.env` files itself — that's the developer's responsibility. The
official guidance:
- Never commit `.env` to git (add to `.gitignore`)
- Rotate keys regularly
- Never hardcode secrets directly in source (Bolt can generate code with keys
  inline — always move these to env vars before deploying)

**Encryption details:** Bolt's documentation states secrets are stored "securely"
but does not disclose the encryption algorithm or key management specifics.

### No Proprietary Backend

Unlike Lovable, Bolt has no built-in backend runtime. If you need a backend:
- Use a third-party API/BaaS (Supabase, Firebase, PlanetScale, etc.)
- Write Edge Functions via the Netlify integration
- Bolt can scaffold the integration code but it runs on Netlify/Supabase — not Bolt

---

## Architecture Summary Diagram

```
[Browser Tab]
  ┌─────────────────────────────────────────────────────┐
  │  WebContainer (WASM)                                │
  │  ┌──────────────┐  ┌──────────────┐                │
  │  │  Rust Kernel │  │  JSH Shell   │                │
  │  └──────────────┘  └──────────────┘                │
  │  ┌──────────────────────────────────────────────┐  │
  │  │  SharedArrayBuffer (filesystem in RAM)        │  │
  │  └──────────────────────────────────────────────┘  │
  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
  │  │ Worker 1 │  │ Worker 2 │  │  Module Resolver │  │
  │  │ (node)   │  │ (npm)    │  │  (ESM + CJS)     │  │
  │  └──────────┘  └──────────┘  └──────────────────┘  │
  └─────────────────────────────────────────────────────┘
         ↕ Service Worker (intercepts fetch)
  ┌─────────────────────────────────────────────────────┐
  │  Preview iframe (live running app)                  │
  └─────────────────────────────────────────────────────┘

         ↕ HTTPS (LLM API calls)
[Anthropic Claude 3.5 Sonnet — edge-hosted]

         ↕ Netlify API (on deploy)
[Netlify CDN — *.netlify.app]
```

---

## Bolt vs Lovable: Key Differences

| Dimension | Bolt.new | Lovable |
|---|---|---|
| Execution environment | Browser (WebContainers/WASM) | Cloud servers |
| Git/version history | None built-in | Every prompt = one commit |
| GitHub sync | Manual | Automatic two-way sync |
| File edit strategy | Full file rewrites (`boltAction`) | Search-replace blocks |
| Supported stacks | Any npm project | React + Vite only |
| Backend | BYO (Supabase, etc.) | Supabase (fully managed) |
| Deployment target | Netlify | Lovable Cloud CDN |
| LLM | Claude 3.5 Sonnet (default) | Gemini 2.5 Flash (default) |
| Cold start | ~seconds (WASM boot) | ~instant (cloud) |
| Offline capable | Partially (after WASM load) | No |

---

## Open Source

Bolt's frontend is **open source**: [github.com/stackblitz/bolt.new](https://github.com/stackblitz/bolt.new)

This means you can read the actual system prompts, tool definitions, and artifact
parser in the repo. The system prompt lives at:
`app/lib/.server/llm/prompts.ts`

WebContainers itself is a closed-source StackBlitz product, though there's a
community fork (bolt.diy) that lets you swap in your own LLM keys and run locally.

---

## Sources

- [How Bolt.new Works — PostHog Newsletter](https://newsletter.posthog.com/p/from-0-to-40m-arr-inside-the-tech)
- [Bolt.new GitHub Repository (open source)](https://github.com/stackblitz/bolt.new)
- [Leaked Bolt.new System Prompt](https://github.com/jujumilk3/leaked-system-prompts/blob/main/bolt.new_20241009.md)
- [StackBlitz + Anthropic Case Study](https://claude.com/customers/stackblitz)
- [Netlify Deployment Integration](https://support.bolt.new/integrations/netlify)
- [Bolt.new LLMs Introduction](https://support.bolt.new/building/intro-llms)
- [Bolt.new Introduction](https://support.bolt.new/building/intro-bolt)
