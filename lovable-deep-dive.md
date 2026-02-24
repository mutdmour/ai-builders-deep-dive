# Lovable — Deep Dive: How It Works Under the Hood

## Overview

Lovable (formerly GPT Engineer) is an AI-powered web app builder that lets users
describe features in natural language and get a fully functional, deployed web app
in return. It reached $100M ARR within 8 months of launch (Dec 2024).

The key insight: it's not a code generator that hands you a zip file. It's a
**continuous edit loop** backed by a real Git repository and a real cloud deployment.

---

## Tech Stack (Always Opinionated)

Lovable only generates one stack — no exceptions:

| Layer | Technology |
|---|---|
| Framework | React + Vite |
| Language | TypeScript |
| Styling | Tailwind CSS |
| Components | shadcn/ui |
| Backend | Supabase (delegated entirely) |
| Auth | Supabase Auth (JWT) |
| Database | Supabase Postgres |
| Server logic | Supabase Edge Functions (Deno) |
| Storage | Supabase Storage |
| Realtime | Supabase Realtime (Postgres CDC) |

No Vue, Next.js, Angular, Svelte, or native mobile. This constraint is intentional —
a narrow, opinionated stack makes the LLM's system prompt tractable and reliable.

---

## How the AI Actually Edits Files

### Leaked System Prompt Insights

Lovable's internal system prompt (leaked Feb 2025) reveals the exact file-editing strategy:

- **Search-replace by default** — not full file rewrites. The LLM outputs targeted
  replacement blocks anchored to existing code.
- **Write-file only for new files** — existing files get surgical, minimal edits.
- **Parallel tool calls** — the system prompt instructs: *"whenever you need to perform
  multiple independent operations, always invoke all relevant tools simultaneously."*
- **Context-aware** — files already loaded into the context window are never re-read
  (they live in a `useful-context` section of the system prompt).
- **Discussion-mode default** — the LLM assumes a planning conversation unless explicit
  action words appear ("implement", "create", "code"). It asks clarifying questions first.

### The Edit Cycle

```
User prompt
  → LLM reads relevant files from context
  → LLM outputs search-replace blocks (or write-file for new files)
  → Lovable's tool layer applies the diffs
  → Files updated in internal project state
  → Auto-commit + push to GitHub
  → Preview re-deployed
```

The LLM never touches the filesystem directly — it emits structured change
descriptions that Lovable's backend interprets and applies.

---

## Version Control: Every Prompt = One Git Commit

### Commit Architecture

Each AI interaction maps to exactly one git commit. There is no batching, staging,
or review step between prompt and commit.

```
Prompt → LLM diffs → apply changes → git commit → git push → preview refresh
```

The commit history in GitHub is a literal transcript of your conversation with the AI.

### Two-Way GitHub Sync

Lovable installs a **GitHub App** (OAuth + webhooks) on your repository:

- **Outbound**: Every edit in Lovable triggers a commit + push to `main`
- **Inbound**: GitHub webhooks notify Lovable when you push from a local clone;
  Lovable pulls and updates its internal state automatically

Lovable's internal editor is the **source of truth** — GitHub is a mirror.

### Constraints

- Only the **default branch** (`main`) is synced. Feature branches are invisible.
- Renaming or moving the GitHub repo breaks the sync permanently.
- Experimental branch support exists in Labs, but is not stable.

### Rollback

The "restore previous version" UI is git under the hood — Lovable reverts to a
prior commit and re-applies it as a new commit on `main`.

---

## Deployment Architecture

### Frontend

Lovable Cloud hosts the built Vite output. The pipeline:

```
git push to main
  → Vite build triggered
  → Static assets deployed to Lovable's CDN
  → *.lovable.app subdomain updated
```

Custom domains with automatic SSL are supported. The underlying CDN provider
is not publicly disclosed, but follows standard JAMstack patterns.

### Backend (All Supabase)

Lovable has no proprietary backend runtime. All server-side logic runs on Supabase:

- **Database operations**: Supabase auto-generated REST/GraphQL from Postgres schema
- **Auth flows**: Supabase Auth handles registration, login, JWTs, OAuth providers
- **Custom server logic**: Lovable generates **Supabase Edge Functions** (Deno) and
  deploys them to your Supabase project via their API
- **Deployed function URL**: `https://[PROJECT_ID].supabase.co/functions/v1/[fn-name]`

When you ask Lovable to "add a Stripe webhook" or "send a welcome email", it:
1. Generates an Edge Function
2. Deploys it to your Supabase project automatically
3. Updates the frontend to call the function endpoint
4. Surfaces runtime error logs back into the chat UI

### Secret Management

Lovable detects when a feature needs credentials (API keys, tokens), prompts you
via UI to enter values, then stores them in **Supabase's encrypted secret manager**.

How it works end-to-end:
1. AI detects a feature needs a secret (e.g., a Stripe integration) and surfaces a UI prompt
2. You enter the value in Lovable's chat UI — it is never written into source code
3. Value is stored encrypted in Supabase's secret manager (Vault)
4. Edge Functions receive secrets as environment variables at invocation time
5. Secret values are never returned to the client browser or embedded in the frontend bundle

**What the LLM sees:** only the secret's *name* (e.g., `STRIPE_SECRET_KEY`), not
the value. It uses the name to reference `Deno.env.get('STRIPE_SECRET_KEY')` in
generated Edge Function code.

**Key guarantee:** secrets never appear in the GitHub repo. They live entirely in
Supabase's infrastructure, decoupled from code.

### Full Architecture Diagram

```
[Lovable Editor / Chat UI]
         |
         | git commit + push (per prompt)
         ↓
[GitHub — main branch]
         |
         | webhook → triggers build
         ↓
[Vite build → Lovable CDN]
   *.lovable.app (frontend)
         |
         | HTTPS API calls
         ↓
[Your Supabase Project]
   ├── Postgres (database)
   ├── Auth (JWTs, OAuth)
   ├── Edge Functions (Deno runtime)
   ├── Storage (files/assets)
   └── Realtime (Postgres CDC → WebSockets)
```

---

## What You Actually Own

Everything Lovable produces is portable:

| Artifact | Portability |
|---|---|
| GitHub repo | Full source, clone and run anywhere |
| Supabase project | Self-hostable, export data anytime |
| Frontend | Standard Vite output, deploy to Netlify/Vercel/Cloudflare Pages |
| Edge Functions | Standard Deno, portable to any Deno runtime |

The "lock-in" is the workflow UX — not the artifacts. You can eject at any time.

---

## Design System Constraints (From System Prompt)

The leaked prompt reveals strict design rules the LLM must follow:

- **Never write custom styles in components** — always use the design system
- Colors and gradients defined in `index.css` and `tailwind.config.ts` using HSL values only
- Use semantic tokens throughout — never explicit classes like `text-white` or `bg-white`
- Create component variants in shadcn components, not ad-hoc overrides

This enforces consistency across AI-generated code and prevents visual drift over
many iterations.

---

## Credit System

Lovable uses a complexity-based credit model:
- Changing a button color → ~0.5 credits
- Adding full authentication → ~1.2 credits

Cost is proportional to how many files the LLM needs to read and edit, and how
many Edge Functions need to be deployed.

---

## Origins

Lovable is a rebrand of the open-source [GPT Engineer](https://github.com/AntonOsika/gpt-engineer)
project (Anton Osika, 2023), which was a CLI tool that generated full codebases from
a single prompt. The commercial product evolved from that proof-of-concept into a
continuous edit loop with hosting, deployment, and a collaborative UI.

---

## Sources

- [Leaked Lovable System Prompt](https://github.com/x1xhlol/system-prompts-and-models-of-ai-tools/blob/main/Lovable/Agent%20Prompt.txt)
- [Lovable GitHub Integration Docs](https://docs.lovable.dev/integrations/git-integration)
- [Lovable Deployment & Hosting Docs](https://docs.lovable.dev/tips-tricks/deployment-hosting-ownership)
- [Lovable Supabase Integration 2.0](https://lovable.dev/blog/lovable-supabase-integration-second-version)
- [Code Surgery: How AI Assistants Make File Edits](https://fabianhertwig.com/blog/coding-assistants-file-edits/)
- [Context Engineering for Coding Agents — Martin Fowler](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html)
- [GPT Engineer → Lovable Evolution](https://lovable.dev/gpt-engineer)
- [How to Build and Scale Full-Stack Apps in Lovable](https://www.productcompass.pm/p/lovable-branching)
