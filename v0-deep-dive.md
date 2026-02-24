# v0 — Deep Dive: How It Works Under the Hood

## Overview

v0 is Vercel's AI-powered app builder. It started (2023) as a pure UI component
generator — describe a component, get a React + Tailwind snippet you could copy into
your project. It has since evolved (2024–2025) into a full-stack coding agent that
generates entire Next.js applications, deploys them to Vercel, and iterates on them
conversationally.

Because it's built by the creators of Next.js and Vercel, the entire system is
vertically integrated: generation → preview → deployment is one seamless pipeline
with zero config.

---

## The Composite Model Architecture

Unlike Lovable (Gemini) or Bolt (Claude), v0 does **not** use a single LLM.
Vercel built a layered pipeline of specialized models:

```
User Prompt
     ↓
[RAG Pre-processing]
  Retrieves relevant docs, UI examples, framework versions, project context
     ↓
[Base Frontier LLM]
  Handles generation for new features / large changes
  v0-1.0-md → Claude Sonnet 3.7
  v0-1.5-md → Claude Sonnet 4
     ↓ (streaming output)
[LLM Suspense — streaming manipulation layer]
  Token replacement, import fixing, library matching (in-flight, <100ms)
     ↓
[AutoFix Model — vercel-autofixer-01]
  Custom fine-tuned (RFT via Fireworks AI)
  Constant error checking during stream + final pass
  10–40x faster than GPT-4o-mini at comparable accuracy
     ↓
[Quick Edit Model] ← routes here for narrow tasks
  Optimized for: text changes, syntax fixes, reordering components
     ↓
Final output
```

### Why a Composite?

LLM-generated code has errors ~10% of the time as a baseline. The pipeline's
three reliability layers (RAG context, LLM Suspense, AutoFix) push that number
down by "double-digit" percentage points according to Vercel's engineering blog.

---

## Dynamic System Prompt + RAG

v0's system prompt is not static. It's assembled at request time:

1. **Intent detection** via embeddings + keyword matching — if the query is about
   AI SDK, React Server Components, a specific Vercel feature, etc., relevant docs
   are injected into the prompt automatically.

2. **Version pinning** — framework versions are injected to override the model's
   training cutoff (e.g., Next.js App Router APIs that changed post-cutoff).

3. **Hand-curated code examples** — a read-only filesystem of patterns (image
   generation, routing, auth flows, etc.) is mounted and sampled via RAG.

4. **Prompt caching** — injected context is kept consistent across requests to
   maximize cache hits and reduce token cost.

5. **Project context** — your uploaded files, git repo contents, and environment
   variable names are embedded and retrieved per query.

This is why v0 often "knows" how to use a library correctly even if it was released
after its training cutoff — the knowledge is injected dynamically, not baked in.

---

## LLM Suspense: Streaming Manipulation

This is one of v0's most unusual technical layers. While the LLM is still generating
(mid-stream), v0 intercepts and rewrites the token stream:

- **URL compression** — long import URLs (hundreds of characters) are replaced with
  short tokens during generation, then reconstructed in the final output. Reduces
  token spend significantly.

- **Import rewriting** — when the LLM references a library export that doesn't
  match exactly (e.g., a renamed Lucide icon), v0 does an embedding search against
  the library's actual exports and rewrites the import within 100ms.

- **Deterministic corrections** — formatting inconsistencies and predictable
  mistakes are pattern-matched and fixed before the stream reaches the user.

The result: errors that would require a follow-up round-trip to the LLM are fixed
silently, in-flight.

---

## The AutoFix Pipeline

`vercel-autofixer-01` is a custom model trained with **Reinforcement Fine-Tuning**
(RFT) via Fireworks AI, built on production failure data from real v0 generations:

- Runs **concurrently** with the base LLM during streaming — it checks the output
  as it arrives, not after
- Performs a **final pass** after generation completes to catch remaining issues
- Handles: missing React provider wrappers (`QueryClientProvider`), broken
  package.json dependencies, import path errors, AST-level structural problems
- Completes within **250ms** end-to-end
- Performance: on par with GPT-4o-mini, 10–40x faster at inference

---

## How the AI Formats Code: The MDX Protocol

From the leaked system prompt (Nov 2024, confirmed by Vercel CTO):

v0 responds in **MDX** — Markdown with embedded executable code blocks. Each block
type has specific behavior:

### React Component Block
~~~
```tsx project="App Name" file="components/button.tsx" type="react"
export default function Component() {
  // ...
}
```
~~~

Rules enforced by the system prompt:
- **Always inline all code** in a single file (for component-level generation)
- **Always write complete code** — no placeholder comments, no truncation
- Must export a `default` function named `Component`
- Must use Tailwind CSS for styling
- Use **shadcn/ui** for components, **Lucide React** for icons
- **Never use indigo or blue colors** unless explicitly asked
- Must generate responsive designs (mobile-first)

### Node.js Executable Block
~~~
```js type="nodejs"
// Runs server-side, output shown in console panel
console.log(result);
```
~~~

### Other Block Types
- `type="html"` — accessible HTML snippets
- Mermaid diagrams
- GitHub Flavored Markdown
- General code (syntax highlighting only, not executed)

### Chain-of-Thought
Before every response, v0 uses `<Thinking>` tags to reason about which block type
is appropriate. This is visible in the UI as a "thinking" state before the response
streams in.

---

## Multi-File & Full-Project Generation

The original v0 generated single self-contained components. The current agent
generates entire Next.js applications with proper file structure:

- Multiple files created in a single generation pass
- Correct Next.js App Router folder conventions (`app/`, `components/`, `lib/`, etc.)
- `package.json` generated and kept in sync with imports
- Environment variables wired up from your connected Vercel project
- Dependency changes (new npm packages) automatically added and installed

The shift from "component generator" to "project generator" was the major 2024
evolution — triggered by adding multi-file output and Vercel project linking.

---

## Deployment Architecture

v0's deployment is natively Vercel — no third-party hosting, no configuration.

### How It Works

```
v0 generates code
     ↓
[Vercel Build System]
  Framework: Next.js (App Router)
  Package manager: Bun (fast installs)
  Output: serverless functions + static assets
     ↓
[Vercel Edge Network (global CDN)]
  Production URL: [project-name].vercel.app
  Unique deployment URL per deploy: [hash]-[project].vercel.app
     ↓
Custom domain (optional, via Vercel dashboard)
```

### First Deploy

Clicking "Publish" for the first time:
1. Creates a new Vercel Project linked to your v0 chat
2. Runs a Bun install + Next.js build
3. Deploys to Vercel's edge network
4. Returns a stable production URL (`[name].vercel.app`)

### Subsequent Deploys

Each deploy triggers a new Vercel deployment with a unique preview URL while
the production URL stays stable. Zero downtime.

### Multiple Chats → Same Project

You can connect multiple v0 chats to the same Vercel Project. Deploying from
any of them updates the same production URL. Useful for feature branches in v0.

---

## Secret Management

v0 uses **Vercel's environment variable system** — the same one used by all
Vercel-deployed apps. Since v0 generates Next.js apps deployed to Vercel, secrets
management is fully native.

### How It Works

**Setting secrets:**
- Added via the Vercel Dashboard → Project Settings → Environment Variables
- Or set via the Vercel CLI: `vercel env add MY_API_KEY`
- v0 can read the *names* of existing env vars in your connected Vercel project
  and wire them into generated code automatically

**Sensitive environment variables (write-only):**
Vercel has a specific "Sensitive" type for high-value secrets:
- Once created, the value is **non-readable** — even project owners can't retrieve it
- Stored in an unreadable (one-way encrypted) format at rest
- Can only be updated by overwriting with a new value
- Available in Production and Preview environments only (not Development)
- Marked with a "Sensitive" tag in the dashboard

**At runtime:**
Secrets are injected as `process.env.MY_KEY` into:
- Next.js API routes (server-side only)
- Edge Functions / Middleware
- Server Components (Next.js App Router)

They are **never bundled into client-side JavaScript** — Next.js enforces this
boundary. To expose a value to the client, it must be prefixed `NEXT_PUBLIC_`,
which signals it's safe to expose (and therefore shouldn't be a secret).

### How v0 References Secrets in Generated Code

When v0 generates an API route that needs a key, it uses:
```typescript
// app/api/chat/route.ts (server-side — safe)
const apiKey = process.env.OPENAI_API_KEY;
```

v0 knows the env var names from your connected project and generates code that
references them by name. The values themselves are never in the conversation
or in the generated files — only the names.

### Env Policy Enforcement

Teams on Vercel Pro/Enterprise can enforce a policy that makes **all** new env
vars in Production/Preview sensitive by default — no plaintext variables allowed.
This is configured under Settings → Security & Privacy → "Enforce Sensitive
Environment Variables."

---

## Export Options (No Lock-in)

v0 outputs are portable in multiple ways:

| Method | Command / Action |
|---|---|
| shadcn CLI | `npx shadcn@latest add <v0-component-url>` |
| ZIP download | Download full project as archive |
| Platform API | Pull `files[]` array programmatically |
| Open in local IDE | Clone generated Vercel project repo |

The `npx shadcn add` flow is particularly elegant — it installs the generated
component directly into your existing project, respecting your existing shadcn
config and Tailwind setup.

---

## The Platform API

Vercel opened v0's generation engine as a REST API, allowing developers to
build their own AI app builders on top of v0's infrastructure:

```typescript
import { v0 } from '@vercel/v0';

const result = await v0.chats.create({
  message: 'Build a dashboard with a sidebar and dark mode toggle',
  // Optional: provide existing files as context
  files: [{ name: 'lib/types.ts', content: '...' }],
});

// result.files: array of { name, content }
// result.demoUrl: live preview link
// result.chatUrl: v0.dev link for further iteration
```

Key capabilities:
- Supply existing files, source code, or git repos as context
- Inject shadcn registry components
- Trigger Vercel deployments programmatically
- Access the same RAG, LLM Suspense, and AutoFix pipeline

---

## Architecture Summary Diagram

```
[User / v0.app UI / Platform API]
              |
              | prompt + project context
              ↓
[RAG Pre-processing]
  Docs, examples, version info, project files
              |
              ↓
[Frontier LLM — Claude Sonnet 3.7/4]
  Generates MDX with code blocks
              |
              | streaming tokens
              ↓
[LLM Suspense Layer]
  URL compression, import rewriting, deterministic fixes
              |
              ↓
[AutoFix Model — vercel-autofixer-01]
  Error detection, provider checks, dep sync
              |
     ┌────────┴────────┐
     | (narrow tasks)  | (large tasks)
     ↓                 ↓
[Quick Edit]     [Full generation]
     └────────┬────────┘
              ↓
[MDX Response with tsx/js blocks]
              |
              | Deploy action
              ↓
[Vercel Build — Next.js + Bun]
              |
              ↓
[Vercel Edge Network]
  *.vercel.app (production)
  [hash].vercel.app (preview per deploy)
```

---

## v0 vs Lovable vs Bolt

| Dimension | v0 | Lovable | Bolt.new |
|---|---|---|---|
| Primary target | UI/full-stack Next.js | Full-stack React apps | Any npm project |
| LLM | Composite (Claude + custom fine-tunes) | Gemini 2.5 Flash | Claude 3.5 Sonnet |
| File editing | MDX blocks (inline full file) | Search-replace diffs | Full file rewrites |
| Error correction | AutoFix + LLM Suspense (in-stream) | None mentioned | None mentioned |
| Execution environment | Vercel cloud | Lovable cloud | Browser (WebContainers) |
| Git / version history | Vercel project per chat | Commit per prompt → GitHub | None built-in |
| Deployment | Native Vercel (1-click) | Lovable Cloud CDN | Netlify (1-click) |
| Backend | Next.js API routes (serverless) | Supabase (fully managed) | BYO |
| Export | npx shadcn, ZIP, Platform API | GitHub sync | ZIP download |
| Open source | No (Platform API available) | No | Yes (frontend) |

---

## Key Design Philosophy

Vercel's CTO on the leaked system prompt:

> "When v0 first came out we were paranoid about protecting the prompt with all
> kinds of pre and post processing complexity. We completely pivoted to let it rip.
> The actual value is in evals, models, and especially UX."

The prompt is not where the value lives. The value is in the composite model
pipeline — particularly LLM Suspense and the AutoFix fine-tune — and the
deep Vercel/Next.js platform integration that makes deployment zero-friction.

---

## Sources

- [How We Made v0 an Effective Coding Agent — Vercel Engineering](https://vercel.com/blog/how-we-made-v0-an-effective-coding-agent)
- [Introducing the v0 Composite Model Family — Vercel](https://vercel.com/blog/v0-composite-model-family)
- [Build Your Own AI App Builder with the v0 Platform API — Vercel](https://vercel.com/blog/build-your-own-ai-app-builder-with-the-v0-platform-api)
- [Leaked v0 System Prompts — Simon Willison](https://simonwillison.net/2024/Nov/25/leaked-system-prompts-from-vercel-v0/)
- [Exploring the v0 System Prompt — DoingWith.ai](https://www.doingwith.ai/articles/exploring-the-v0-dev-system-prompt)
- [v0 Deployments Docs](https://v0.app/docs/deployments)
- [v0 What is v0 Docs](https://v0.app/docs)
- [Maximizing Outputs with v0 — Vercel](https://vercel.com/blog/maximizing-outputs-with-v0-from-ui-generation-to-code-generation)
