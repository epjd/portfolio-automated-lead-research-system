# Multi-Agent Lead Signal Pipeline (n8n)

> 🧭 **Portfolio project by Elijah Peh** - Solutions Engineer / Forward-Deployed Engineer.
> - Connect on [LinkedIn](https://www.linkedin.com/in/elijah-peh-3b5bb8ba/).

A **3-stage agentic AI pipeline, built end-to-end on [n8n](https://n8n.io)**, that continuously monitors industry and regulatory sources, uses AI to spot and qualify high-value sales signals, and hands each qualified opportunity to the business's BD team with outreach content already drafted - ready to review, tweak, and send.

**The problem it solves:** the business already has contact details for its target companies. What it *didn't* have was time - someone would otherwise have to manually check dozens of news sources and regulatory databases every day to catch new product launches and filings. This system does that watching automatically, so Business Development (BD) can **get ahead of the curve** and reach out the moment a relevant signal appears.

**Human in the loop by design:** the pipeline automates the tedious part - monitoring, qualification, and drafting - but stops short of contacting anyone. Every qualified lead lands in a BD reviewer's inbox, where a person checks the signal, edits the drafted content, and decides whether to pursue.

**At a glance:** 
- 3 independent workflows + 1 shared image-generation sub-workflow
- 15+ monitored data sources
- 20+ prospects qualified per run
- ~50% less manual research effort
- self-hosted with parallel error monitoring

> Built for an enterprise client in regulated manufacturing. This is a sanitized, portfolio version - client names, internal infrastructure, credentials, and proprietary scoring rules have been removed or generalized.

---

## Why it's interesting

Most "lead-gen automation" is a single scrape-and-blast script that fires cold emails at strangers. This system is the opposite: it's a **signal-intelligence layer** for a BD team that already knows *who* its customers are but can't watch every source for *when* to reach out. It models the business's **product development lifecycle** and runs a dedicated pipeline for each commercial stage, so the signals it looks for, the qualification criteria it applies, and the content it drafts are all stage-appropriate - and a human always makes the final call.

It's a study in three things I care about as a solutions engineer:

- **Reliability in production** - error handling, deduplication, and status gating so nothing double-sends or silently fails.
- **Cost discipline** - token consumption is the main cost driver, so the architecture actively minimizes unnecessary LLM calls.
- **Designed to scale** - a documented roadmap from a built-in datastore to external DBs, semantic dedup, and CRM integration.

---

## Built on n8n

The entire system is orchestrated in **n8n** (self-hosted). Every stage is a native n8n workflow, and I used n8n as the connective tissue holding the whole pipeline together:

- **Schedule triggers** drive each stage independently, so runs can be staggered to control load and cost.
- **Data tables** (n8n-native) hold the lead list, raw source data, evaluation results, and generated content - separated by workflow and data type.
- **Code nodes** handle HTML cleaning, article extraction, and the deduplication/token-optimisation logic that keeps AI calls down.
- **HTTP request nodes** integrate the scraping API and regulatory APIs directly.
- **AI agent nodes** run the evaluation and content-generation logic per stage.
- **A dedicated error-handling workflow** runs in parallel and watches the failure-prone nodes across all three stages.

The result is a production system a non-engineer can inspect and operate visually, without touching code.

---

## Architecture

Three independent workflows share one architectural pattern but differ in signal logic, data sources, AI evaluation criteria, and content output.

| Workflow | Stage focus | Signal type | Example sources |
|---|---|---|---|
| **Early** | New product launches & announcements | "The Value" | Company PR, industry news publications |
| **Mid** | Regulatory submissions & technical file activity | Problem-focused | Company PR, regulatory databases (APIs) |
| **Late** | Active trials & confirmed launch dates | Social proof / commercial | Company PR, industry news publications |

### Common pipeline

```
[Schedule Trigger]
        │
        ▼
[Lead List] ── Filter: Status = "Active"
        │
        ▼
[Source Scraping / API Query]
(web scraping for news sources / direct API for regulatory sources)
        │
        ▼
[HTML Cleaning & Article Extraction] ── code node
        │
        ▼
[Deep Article Extraction] ── web sources only
        │
        ▼
[Deduplication & Token Optimisation] ── code node
        │
        ▼
[AI Evaluation Agent] ── stage-specific qualify / disqualify logic
        │
        ├── Disqualified → stop
        └── Qualified →
                    [Research Store]
                            │
                    [Content Generation AI Agent]
                            │
                    [Content Store]
                            │
                    [Email Alert → internal BD reviewer]
                    (qualified lead + drafted outreach content)
                            │
                    [Update Status = "Alerted"]
                            │
                            ▼
                    Human in the loop:
                    BD reviews the signal & drafted content,
                    edits as needed, and decides whether to
                    pursue the lead using existing contact details.


[Email Alert → internal BD reviewer]
    (qualified lead + drafted outreach content
     + "Generate image options" button per variation)
            │
    [Update Status = "Alerted"]
            │
            ▼
    Human in the loop:
    BD reviews the signal & drafted content,
    edits as needed, and decides whether to
    pursue the lead using existing contact details.
            │
    (optional, human-triggered)
            ▼
[Webhook: Generate Image Options]
            │
[Duplicate-request check] ── already requested → styled status response
            │
[Creative-Direction AI Agent] ── 3 stage-calibrated concepts
            │
[Image Generation ×3]
            │
[Follow-up email: 3 image options as attachments]
```

**Mid-stage variation:** runs multiple evaluation agents (one per source) and consolidates them through a unified qualification filter before content generation, with raw data stored per source.

---

### Visual creative generation (human-triggered)

[](#visual-creative-generation-human-triggered)

Beyond the qualification and drafting pipeline, a **dedicated sub-workflow** lets a BD reviewer request AI-generated visual creative for any drafted content variation, directly from the alert email — no separate tool or login required.

**How it works:**

1. Each drafted content variation includes a "Generate image options" button, calling a webhook that carries the variation's ID and pipeline stage.
2. The webhook routes to the correct stage's data, retrieves the underlying content context (headline, body copy, or concept brief depending on stage), and passes it to a **creative-direction AI agent**.
3. The agent proposes three distinct, stage-calibrated image directions (e.g. early-stage signals favor editorial/innovation framing, late-stage favors momentum/decision framing) and passes each to an image-generation model.
4. The three resulting images are attached to a follow-up email as PNG files alongside their creative rationale, ready for the reviewer to pick from.

**Design constraints solved:**

- **Duplicate-request protection** - a status flag per content row ensures repeated clicks don't trigger repeated (costly) generations; the reviewer instead sees a clear "already generated - check your inbox" response.
- **Email client compatibility** - inline/embedded images were tested but proved unreliable across email clients (notably Outlook); images are delivered as standard attachments instead, which render consistently everywhere.
- **Styled webhook responses** - rather than plain-text confirmation, the webhook returns lightweight styled HTML so the click-through experience (generating / already requested / error states) is visually clear and on-brand, not a raw browser default.

This turns a static content-review email into a lightweight, self-serve creative request tool - still fully human-gated, since nothing is generated without an explicit BD click.
---

## Qualification logic

**Shared across all workflows**

- **Lead filter** - only companies with `Status = "Active"` are processed.
- **Deduplication** - articles already in the research store are skipped before they reach an AI agent.
- **Qualification** - AI agents score each signal against stage-specific criteria.
- **Content gate** - only qualified leads get generated content.
- **Alert gate** - only new, unreviewed content rows trigger a BD alert, so nothing is surfaced twice.

**Stage-specific signals** (generalized)

| Stage | Qualifies on | Disqualifies on |
|---|---|---|
| **Early** | New, relevant product launch matching target category | Out-of-scope product types, stale news |
| **Mid** | Regulatory submission activity with high packaging relevance and high lead tier | Out-of-scope devices, low-relevance signals |
| **Late** | Active trial with evidence, or confirmed launch with an explicit date | Announcements without a date, out-of-scope leads, stale news |

---

## Tech stack

- **Workflow automation:** n8n (self-hosted)
- **Web scraping:** managed scraping API (handles JS-rendered pages & anti-bot protection)
- **AI agents:** OpenAI GPT-4.1 - powers all evaluation and content-generation agents
- **Email delivery:** transactional email API
- **Data:** workflow-scoped tables (raw source data, evaluation results, generated content), separated by workflow and data type

> **Cost note:** token consumption is the primary cost driver. The deduplication step is what keeps it in check - filtering already-processed articles out *before* they reach an agent.

---

## Reliability: error handling

A dedicated error-handling workflow runs in parallel across all three stages and monitors the failure-prone nodes:

- scraping HTTP requests,
- AI evaluation agents,
- AI content-generation agents.

On failure, it sends an automated notification to the workflow owner - no manual log-watching required.

---
**Lessons learned in production:**

- **Pass-by-reference over implicit context** - relying on the "current item" (`$json`) after a node that doesn't guarantee item-to-item traceability (e.g. an HTTP response from a transactional email API) can silently drop upstream fields. Referencing a specific upstream node by name (`$('NodeName')`) proved more robust wherever branch logic downstream depended on fields set earlier in the flow.
- **Data schema drift across near-identical branches** - Each stage's data table used different column names for conceptually identical fields, which was hidden by normalization early in the flow but resurfaced whenever a later node accidentally bypassed the normalized shape. Solved by tracing every downstream reference back to its normalized source rather than assuming a shared shape across stages.
- **Silent routing failures** - a Switch/Router node with no matching branch fails quietly rather than erroring, which can look identical to "no data" upstream. Adding explicit fallback branches (or defensive expression scoping) surfaces this instead of leaving it to look like a dead end.

---

## Scaling roadmap

Designed for the initial scope, with a clear path forward as volume grows:

- **More sources / companies** → higher scraping and token costs; monitor usage, stagger or batch triggers, and keep deduplication effective as research tables grow. Large workflows may need splitting or higher execution limits.
- **Feedback loop** → add an outcome column (Converted / No Response / Bounced), optionally backed by a vector DB for semantic dedup and similarity search, and feed conversion data back into agent prompts to refine scoring over time.
- **Data out of the built-in store** → migrate to an external database (e.g. PostgreSQL / Supabase / Airtable) for richer querying, dashboards, and reporting - only read/write nodes change.
- **CRM integration** → route qualified-lead alerts into a CRM or pipeline view instead of email, keeping the BD review step but giving it a richer home.

---

## What I owned

Solution design, workflow implementation, AI agent prompt design and qualification logic, scraping/API integrations, cost optimization, and error handling - end to end.
- **Image generation:** OpenAI image model, 1536×1024 output, delivered as email attachments (not inline, for cross-client rendering reliability)
