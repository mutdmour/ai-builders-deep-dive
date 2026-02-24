# AI Builders — Deep Dives

Technical deep dives into how today's AI coding tools and app builders work under the hood: sandboxes, LLM pipelines, file editing strategies, deployment, secrets management, and more.

## Docs

| Tool | Type | Key Tech |
|---|---|---|
| [Lovable](./lovable-deep-dive.md) | Cloud app builder | Supabase, Gemini, GitHub sync, search-replace edits |
| [Bolt.new](./bolt-new-deep-dive.md) | Browser IDE | WebContainers (Rust/WASM), Claude, Netlify |
| [v0](./v0-deep-dive.md) | Cloud app builder | Composite model (Claude + AutoFix fine-tune), LLM Suspense, Vercel |
| [Cursor](./cursor-deep-dive.md) | Local IDE (VS Code fork) | Fast Apply (speculative edits), Shadow Workspace, Turbopuffer |
| [OpenHands](./openhands-deep-dive.md) | Agent framework | Event sourcing, CodeAct, Docker sandbox, SecretRegistry |
| [Codex Web](./codex-web-deep-dive.md) | Cloud async agent | codex-1 (o3 fine-tune), two-phase container, App Server (JSON-RPC) |
| [n8n](./n8n-deep-dive.md) | Self-hosted workflow automation | Node execution engine, expression sandbox, Bull+Redis queue, plugin nodes |

## What's Covered per Tool

Each doc covers:
- **Architecture** — how the system is structured end-to-end
- **LLM pipeline** — which models, how they're composed, any custom fine-tunes
- **File editing strategy** — how the AI's output gets applied to actual files
- **Execution environment** — where code runs (browser, container, local, cloud)
- **Deployment** — how apps get hosted and published
- **Secret management** — how credentials are handled (and where the risks are)
- **System prompt internals** — where available from leaks or open-source repos

## Comparison

| Tool | File Editing | Execution Env | Secrets | Open Source |
|---|---|---|---|---|
| Lovable | Search-replace | Cloud | Supabase Vault | No |
| Bolt.new | Full file rewrite | Browser (WASM) | Built-in UI (server-side only) | Partial |
| v0 | MDX blocks (full file) | Cloud (Vercel) | Vercel sensitive env vars | No |
| Cursor | Speculative edits (Fast Apply) | Local machine | None built-in | No |
| OpenHands | CodeAct (executable code) | Docker container | SecretRegistry + output masking | Yes (MIT) |
| Codex Web | Agent loop (bash + file tools) | Managed cloud container | Phase-gated (setup only) | Partial |
| n8n | Node execution engine | Local / Docker / Cloud | AES-256-GCM + external vaults | Yes (fair-code) |

## Sources

Each doc links to primary sources: official engineering blogs, leaked system prompts, research papers, and open-source code where available.
