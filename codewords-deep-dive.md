# Codewords — Deep Dive: How It Works Under the Hood

## Overview

Codewords (by Agemo) is a **chat-native workflow automation platform** — an AI-first alternative to Zapier, Make, and n8n where users describe what they want in plain language and the system generates, deploys, and runs actual executable code. There is no visual node editor; the primary interface is a chat conversation with an AI assistant named Cody.

Under the hood, Codewords uses **Dunia**, Agemo's proprietary neurosymbolic reasoning engine, to convert natural language intent into Python code that runs in isolated serverless sandboxes. Integrations with 2,700+ apps are handled by Pipedream's Connect platform.

The company (Agemo) is a London-based startup founded by Aymeric Zhuo and Osman Ramadan, backed by Fly Ventures and FirstMinute Capital (£3.2M seed). The team brings experience from Microsoft, Palantir, Meta, PolyAI, Cambridge, and the Alan Turing Institute.

---

## The Core Architecture

### High-Level Stack

```
User (natural language) → Cody AI chat interface
                                ↓
                      Dunia reasoning engine
                                ↓
                      Python code generation
                                ↓
                   Serverless execution sandbox
                                ↓
                    Pipedream integration layer
                                ↓
                    2,700+ external APIs/services
```

### Design Philosophy

The key architectural decision that differentiates Codewords from Zapier/Make/n8n: **generate code, not configurations**.

Traditional automation platforms represent workflows as directed graphs of predefined "action nodes" (send email → filter → update row). This is visually intuitive but architecturally limiting — the expressiveness is bounded by what node types exist.

Codewords instead generates **actual Python source code** that can express arbitrary logic: loops, conditionals, data transformations, error handling, stateful tracking. The generated code is visible to users and can be modified or exported.

---

## The LLM Pipeline: Dunia Reasoning Engine

### Why Not Pure LLMs?

Agemo's technical position is that standard LLMs are insufficient for reliable code generation because they reproduce patterns from training data rather than reasoning about requirements:

> "LLMs by themselves aren't really reasoning about the world — they're reproducing patterns from their training data."

Agemo's answer is **Dunia**, a neurosymbolic system that pairs a neural component (LLMs for natural language understanding) with a symbolic component (formal reasoning engines for verification and constraint satisfaction).

### Neurosymbolic Architecture

**Neural layer**: LLMs understand the user's natural language intent, identify relevant integrations, and propose code structure.

**Symbolic layer**: Formal reasoning engines validate decisions — checking that generated code satisfies constraints before execution. This is where Codewords differs most from competitors: it uses automated verification rather than relying on the LLM to be correct by chance.

**RLVR (Reinforcement Learning with Verifiable Rewards)**: The system learns from execution results. Code either runs successfully or fails — a clean binary signal. This objective feedback trains the model what "correct" looks like without requiring human evaluators for every generated function.

```
User describes automation
        ↓
Dunia decomposes into concrete steps (symbolic planning)
        ↓
LLM generates Python code for each step
        ↓
Automated checks run (syntax, execution, integration test)
        ↓
Binary reward signal (✓ pass / ✗ fail)
        ↓
RLVR updates model weights over time
```

### Code Generation Process

1. **Specification**: User describes automation in chat ("every morning, check if any GitHub PRs were opened overnight and post a summary to Slack")
2. **Planning**: Dunia decomposes into discrete steps with defined inputs/outputs
3. **Integration selection**: Identifies which Pipedream connectors to use (GitHub, Slack)
4. **Code generation**: Produces Python (FastAPI-based) implementing each step
5. **Validation**: Automated testing runs against real data
6. **Deployment**: Deploys to serverless infrastructure; returns webhook URL or configures schedule
7. **Learning**: Success/failure feeds back into RLVR training pipeline

---

## Code Generation: What Gets Generated

### Language and Framework

Generated code is **Python** using **FastAPI** patterns, deployed as serverless functions.

```python
# Example generated automation handler
from fastapi import Request
from typing import Optional
import httpx

async def main(request: Request):
    # Fetch GitHub PRs opened in last 24 hours
    gh_response = await fetch_github_prs(
        repo="myorg/myrepo",
        since=yesterday_iso()
    )

    if not gh_response.prs:
        return {"status": "no_new_prs"}

    # Format and send to Slack
    message = format_pr_summary(gh_response.prs)
    await send_slack_message(
        channel="#engineering",
        text=message,
        token=get_secret("SLACK_BOT_TOKEN")
    )

    return {"status": "ok", "prs_summarized": len(gh_response.prs)}
```

Why Python:
- LLMs have the strongest Python training data → most reliable generation
- Massive ecosystem for APIs, data processing, ML
- Serverless-ready (AWS Lambda, Google Cloud Functions support Python natively)
- Generated code is readable by non-developers, reducing "black box" concern

### Code Visibility and Export

Users can view the underlying Python through the chat interface. This is a deliberate trust mechanism — users can see exactly what their automation does, modify it, or export it for use elsewhere. This reduces vendor lock-in compared to systems that keep the implementation fully opaque.

---

## Execution Environment

### Serverless Architecture

Each automation runs as an **isolated serverless function**:

```
Automation triggered (webhook / schedule / manual)
        ↓
Serverless runtime allocates isolated compute
        ↓
Python function executes
        ↓
Results stored; logs captured
        ↓
Compute released (pay-per-execution)
```

**Properties:**
- Zero idle cost — resources only consumed during execution
- Automatic scaling — function invocations are fully parallel
- No infrastructure management — users never provision or configure compute
- Failed runs only charge for completed steps (partial execution billing)

### State Persistence

Cross-execution state is built in — automations can track which records they've already processed:

```python
# Track processed IDs across runs
processed = get_state("processed_pr_ids") or set()
new_prs = [pr for pr in all_prs if pr.id not in processed]
# ... process new_prs ...
update_state("processed_pr_ids", processed | {pr.id for pr in new_prs})
```

This avoids the common automation anti-pattern of reprocessing the same records on every run without needing an external database.

---

## Integration Layer: Pipedream Connect

### The Foundation: 2,700+ Pre-Built Connectors

Codewords doesn't build or maintain API integrations in-house. Instead, it's built on top of **Pipedream Connect** — Pipedream's embeddable integration platform that provides:

- 2,700+ pre-built API connectors
- Managed OAuth and credential refresh
- API proxy (requests route through Pipedream's servers)
- Type-safe client libraries per service

When the AI generates code that calls Gmail, Slack, Salesforce, or any other service, it uses Pipedream's connector — not raw HTTP calls. Pipedream handles authentication transparently.

### Authentication Flow

```
User connects a service (e.g., Gmail):
1. Codewords/Pipedream initiates OAuth flow
2. User authorizes in browser popup
3. OAuth tokens stored in Pipedream's credential vault
4. At execution time, Pipedream injects valid access token
5. Token refresh handled automatically by Pipedream
6. Generated code never sees the raw token
```

This means credential management — including token rotation, expiry, and refresh — is entirely delegated to Pipedream's infrastructure.

### Integration Categories

| Category | Examples |
|---|---|
| Communication | Slack, Gmail, Discord, WhatsApp, Telegram |
| CRM/Sales | Salesforce, HubSpot, Pipedrive |
| Project management | Notion, Asana, ClickUp, Linear, Jira |
| Data/storage | Google Sheets, Airtable, Supabase, PostgreSQL |
| Dev tools | GitHub, GitLab, Webflow |
| E-commerce | Shopify, WooCommerce |
| Marketing | Mailchimp, ConvertKit, ActiveCampaign |
| AI services | OpenAI, Anthropic, Google Gemini |

---

## Triggering and Scheduling

### Trigger Types

**Scheduled (cron-style):**
```
"Run every day at 9am"
"Every Monday morning"
"On the 1st of each month"
```

**Webhook:**
Generated automations receive a unique webhook URL. Any service that supports HTTP POST can trigger the automation. Cody auto-generates the webhook URL and provides connection instructions.

**Event-based:**
- Slack reactions, new messages
- Incoming emails
- Form submissions
- Database row changes (via polling or CDC)

**Manual:** Run from the UI for testing or one-off execution.

### Deployment

Deployment is fully automatic — when a user approves the generated automation, Cody:
1. Packages the Python code
2. Registers triggers (webhook URLs, cron schedules)
3. Deploys to serverless infrastructure
4. Returns confirmation with webhook URL or schedule details

No CI/CD, no YAML, no Docker required on the user's side.

---

## Secret Management

### Credential Architecture

Credentials are stored in two places depending on type:

**OAuth tokens (via Pipedream):**
- Stored in Pipedream's encrypted vault
- Never exposed to generated code as raw values
- Injected at execution time by Pipedream's proxy layer
- Automatically refreshed before expiry

**API keys and other secrets:**
- Stored encrypted in Codewords' own infrastructure
- Referenced by name in generated code (e.g., `get_secret("OPENAI_API_KEY")`)
- Injected at execution time, not stored in the code itself

### Key Security Properties

- API keys are never stored in generated Python source code
- OAuth flow tokens are managed entirely by Pipedream, not by Codewords
- Credentials cannot be viewed after initial entry (write-only)
- Executions run in isolated sandboxes — one automation cannot access another's secrets

### Risk Model

Unlike Replit (where the AI agent has full environment access) or Cursor (where secrets are in the local filesystem), Codewords' architecture keeps the LLM separated from credential values. The AI generates code that _references_ secrets by name; the actual values are only resolved at serverless execution time, in isolation.

---

## Comparison to Competitors

| Dimension | Codewords | Zapier | Make (Integromat) | n8n |
|---|---|---|---|---|
| **Interface** | Chat (natural language) | Visual node editor | Visual flowchart | Visual node editor |
| **Code generation** | Yes (Python) | No | No | Partial (JS expressions) |
| **Underlying representation** | Executable Python | JSON action graph | JSON scenario | JSON workflow |
| **AI reasoning** | Neurosymbolic (Dunia) | AI step suggestions | AI step suggestions | Partial |
| **Integrations** | 2,700+ (Pipedream) | 7,000+ (own) | 1,000+ (own) | 400+ (own) |
| **Self-hostable** | No | No | No | Yes (Apache 2.0) |
| **Execution** | Serverless (managed) | Serverless (managed) | Cloud (managed) | Self-hosted or cloud |
| **State persistence** | Built-in | Workarounds needed | Storage module | DB node |
| **Code visibility** | Yes (view + export) | No | No | Yes (JSON) |
| **Open source** | No | No | No | Yes |
| **Pricing** | Credit-based ($0/$20/$100) | Task-based ($19.99/$49+) | Operation-based ($9/$16+) | Execution-based ($20+) |

### Codewords' Key Advantage

The chat interface dramatically lowers the barrier to entry for non-technical users. Building a workflow in Zapier requires learning which "triggers" and "actions" exist and how to configure filter conditions. In Codewords, you describe what you want in plain English and the system figures out the implementation.

The trade-off: because it generates code rather than configuring predefined nodes, there's less user control over exactly _how_ integrations are called. Power users who want precise API control may prefer n8n or direct code.

---

## AI Model Integration

### First-Class LLM Support

As a platform designed around AI, Codewords has built-in, pre-authenticated connectors for major AI APIs:

- **OpenAI**: GPT-4, GPT-4o, Assistants API, image generation
- **Anthropic**: Claude 3.5 Sonnet, Claude 3 Haiku
- **Google**: Gemini Pro, Gemini Flash

Generated code using these APIs handles:
- API authentication (key injection)
- Request formatting and response parsing
- Rate limiting and retry logic
- Token limit management via automatic text chunking

Example generated AI workflow step:

```python
async def analyze_sentiment(text: str) -> dict:
    response = await anthropic_client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=100,
        messages=[{"role": "user", "content": f"Sentiment of: {text}"}],
    )
    return {"sentiment": response.content[0].text}
```

### Web Scraping Capabilities

Two mechanisms for web data extraction:

**Firecrawl integration:**
- Handles JavaScript-rendered pages
- Returns clean Markdown or structured JSON
- Full-page screenshots
- ~2–5 second processing per page

**Chrome extension:**
- Point-and-click extraction rule definition
- Visual selector builder
- Converted to reusable extraction steps in automations

---

## The Dunia Research Background

### ARC-AGI Work

Agemo released **arcsolver**, an open-source Python library that achieved 85% accuracy on the ARC (Abstraction and Reasoning Corpus) training tasks — a benchmark designed to test AI's ability to generalize from few examples, which pure LLMs struggle with significantly.

The approach used object-centric grid modeling with Pydantic schemas and symbolic reasoning over discrete grid transformations. This research directly informs Dunia's approach to decomposing user intent into verifiable sub-problems.

### Process-Guided Search

Rather than sampling multiple completions from an LLM and picking the best (expensive), Agemo's research focuses on **process-guided search** — navigating a structured solution space using reinforcement learning:

- Q-learning for exploration vs. exploitation tradeoffs
- Direct preference optimization for vision models
- Skill acquisition: the system learns new reasoning primitives autonomously rather than having them hardcoded

### Long-Term Vision

Agemo's stated goal: "Code will become a forgotten artefact." Their view is a transition from "Software-as-a-Service" to "Service-as-a-Software" — users describe desired behavior, AI handles the full software pipeline. Codewords is the current product expression of this research direction.

---

## Architecture Summary Diagram

```
[User — natural language]
        ↓ chat
[Cody AI Assistant]
        ↓
[Dunia Reasoning Engine]
  ├── Neural: LLM (intent understanding)
  ├── Symbolic: formal plan verification
  └── RLVR: learning from execution feedback
        ↓
[Python code generation (FastAPI-based)]
        ↓
[Automated testing + validation]
        ↓
[Serverless deployment (automatic)]
  ├── Webhook trigger → unique HTTPS endpoint
  ├── Cron schedule → time-based execution
  └── Event trigger → service-specific listener
        ↓
[Serverless execution sandbox]
  ├── Isolated compute per run
  ├── State persistence (cross-run tracking)
  └── Secret injection (at execution time)
        ↓
[Pipedream Connect integration layer]
  ├── 2,700+ pre-built connectors
  ├── OAuth token management
  └── Authenticated API proxy
        ↓
[External services — Slack, GitHub, Gmail, Salesforce, ...]
```

---

## Pricing and Limits

### Credit-Based Pricing

| Plan | Monthly Cost | Credits | Notes |
|---|---|---|---|
| Free | $0 | $5 one-time | ~12 builds, ~2,500 executions |
| Pro | $20 | $30 + 50% bonus = $45 | ~50 builds, ~10,000 executions |
| Max | $100 | $200 + 100% bonus = $400 | ~250 builds, ~50,000 executions |
| Enterprise | Custom | Custom | SLAs, dedicated support |

### Credit Consumption

| Action | Cost |
|---|---|
| Build automation | $0.50–$2 |
| Edit automation | $0.10–$1 |
| Run automation | $0.00001–$1 (depends on complexity) |

Failed runs only charge for completed steps. No idle charges.

---

## Open Source Status

Codewords itself is **proprietary/closed-source**. Agemo has open-sourced:
- **arcsolver**: Python library for ARC-AGI challenge (symbolic grid reasoning)
- Research papers on neurosymbolic AI and RLVR

The integration layer (Pipedream) is partially open source (Pipedream's component registry is public).

---

## What Makes Codewords Architecturally Distinct

1. **Code generation over configuration** — generating executable Python rather than configuring predefined node types gives the system unbounded expressiveness. Any logic that can be written in Python can be an automation.

2. **Neurosymbolic reasoning over pure LLMs** — Dunia's symbolic verification layer reduces hallucination risk for code generation. Automated checks (execution results, linting, integration tests) provide clean reward signals for RLVR training.

3. **Pipedream as integration infrastructure** — delegating 2,700+ integrations to Pipedream means Codewords doesn't need to maintain API connectors. Managed OAuth also means Codewords never handles raw credentials for third-party services.

4. **Chat-native as primary interface** — not a visual editor with AI bolted on. The entire product is designed around describing intent in plain language, making it accessible to non-technical users without sacrificing code-level expressiveness.

5. **State persistence without external databases** — built-in execution state means automations can track processing history across runs without users needing to provision a database or manage state externally.

---

## Sources

- [Codewords — Main Site](https://codewords.ai/)
- [Codewords Documentation](https://docs.codewords.ai/)
- [Introducing Codewords Beta](https://codewords.ai/blog/introducing-codewords-beta-the-idea-to-software-platform)
- [Codewords Features](https://codewords.ai/blog/codewords-features)
- [Introducing Agemo](https://codewords.ai/blog/introducing-agemo)
- [Summer of ARC-AGI Part 1](https://codewords.ai/blog/summer-of-arc-agi-part-1)
- [Codewords Pricing](https://docs.codewords.ai/get-started/pricing)
- [Pipedream Connect Platform](https://pipedream.com/connect)
- [Agemo Exits Stealth (Lambham)](https://www.lambham.com/post/agemo-exits-stealth-with--88m-to-build-ai-reasoning-software/)
- [RLVR Explained — Promptfoo](https://www.promptfoo.dev/blog/rlvr-explained/)
