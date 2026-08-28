# True Digital - AI Growth Agent

An autonomous prospecting and opportunity-intelligence system that finds businesses with visible, fixable growth gaps - weak SEO, unclaimed Google Business Profiles, missing marketing automation - scores the commercial opportunity, and produces a human-approved, personalized outreach draft. Built to give a solo agency operator the prospecting throughput of a small BDR team, on free/low-cost infrastructure.

## Problem

Small and mid-sized businesses rarely self-diagnose their own digital growth gaps, and generic cold outreach doesn't work on them. Manual prospecting - researching a company, identifying what's actually broken, estimating a believable ROI, and writing a personalized first message - takes 30–45 minutes per prospect by hand. That caps outbound volume at a level too low to build a pipeline. This system collapses that to under 3 minutes of human review time per opportunity, with every send still gated by human approval.

## Architecture

Two orchestrators coordinate five specialized sub-agents. Orchestrator 1 owns discovery, enrichment, and CRM writes; Orchestrator 2 owns analysis, scoring, and outreach drafting. Splitting them keeps each orchestrator's context window focused - discovery never sees pricing/ROI logic, and analysis never handles raw scraping.

```mermaid
flowchart TD
    A[Schedule Trigger - daily<br/>or Manual Trigger] --> B[Select Target Segment<br/>auto-rotates city + industry]
    B --> C[Firecrawl Search]
    C --> D{Filter directory/social<br/>sites out}
    D --> E[Firecrawl Scrape]
    E --> F[Compute Site Signals]
    F --> G[Information Extractor<br/>Groq - primary]
    G -- error --> G2[Information Extractor<br/>OpenRouter - fallback]
    G --> H[Prepare Lead Record]
    G2 --> H
    H --> I[Orchestrator 2:<br/>Opportunity Intelligence]

    subgraph I [ ]
        I1[Research Agent]
        I2[Automation Consultant]
        I3[Solution Architect]
        I4[ROI Calculator]
        I1 & I2 & I3 & I4 --> J[Merged Opportunity<br/>Intelligence Report]
        J --> K[Outreach Writer]
    end

    K --> L{Company already<br/>in HubSpot?}
    L -- yes --> M[Skip - dedup]
    L -- no --> N[Create Company + Deal]
    N --> O[Log Opportunity Report<br/>as HubSpot Task]
    O --> P[Slack: per-opportunity<br/>review notification]
    O --> Q{Contact email found?}
    Q -- yes --> R[Create Gmail draft<br/>- never auto-sent]
    Q -- no --> S[Skip draft]
    R --> T[Slack: batch complete]
    S --> T
```

Every LLM call in the pipeline (6 call sites total: extraction + 4 analysis sub-agents + outreach drafting) runs on Groq by default and fails over to OpenRouter automatically on error or rate limit - this has caught real Groq rate-limit failures in production without stalling the batch.

## Tech Stack

| Layer | Tool |
| --- | --- |
| Orchestration | n8n (Cloud) |
| Search + scraping | Firecrawl |
| Primary LLM | Groq (`openai/gpt-oss-20b`) |
| Fallback LLM | OpenRouter (`nvidia/nemotron-3.5-lightning:free`) |
| Structured output | LangChain Information Extractor nodes, JSON schema enforced per agent |
| CRM | HubSpot (company / contact / deal / task objects) |
| Outreach | Gmail API - draft-only scope, send permission deliberately not granted |
| Notifications | Slack |

## Sub-Agents

| Agent | Responsibility |
| --- | --- |
| Research Agent | Extracts SEO score, load speed, GBP/LinkedIn/Instagram presence from scraped site signals |
| Automation Consultant | Reasoning chain: Observation → Reasoning → Problem → Primary Pain Point, grounded only in the Research Agent's data |
| Solution Architect | Maps the pain point to a service-catalog pattern (6-industry automation pattern library) |
| ROI Calculator | Produces a conservative hours-saved and ROI range - never a single point estimate |
| Outreach Writer | Drafts a 5-part, no-placeholder first-touch message referencing only verified data points from the report |

## Sample Opportunity Intelligence Report (redacted)

```
Company: [Example Skin Clinic]
Confidence Score: 80/100

BUSINESS OVERVIEW
SEO Score: 100/100
Load Speed: 3.5s
Google Business Profile Claimed: false

PAIN POINT
Observation: GBP has not been claimed.
Reasoning: Without a claimed Google Business Profile, the clinic appears
less in local search results, reducing potential patient inquiries.
Primary Pain Point: Unclaimed Google Business Profile limiting local
search visibility.

RECOMMENDED AUTOMATION
Opportunity: Claim and optimize Google Business Profile, integrate
Lead-Capture Automation to convert increased visibility into booked
appointments.
Priority: 4

ROI
Estimated Hours Saved/Month: 15
Estimated ROI: 2 to 5 additional booked calls/month

NEXT ACTION
20-minute free automation audit call
```

## Proof of work - a real run (28 Aug 2026)

Pulled directly from production executions, not a staged demo. The Schedule Trigger fired at 9:00 AM IST; Orchestrator 1 found 2 new leads, and each one ran the full Orchestrator 2 chain automatically:

| Lead | SEO score | Primary pain point | ROI estimate | Draft ready in |
| --- | --- | --- | --- | --- |
| Skinshine Skin & Hair Clinic | 50 | No Google Business Profile claimed | 7 hrs/mo saved, 1–5 additional bookings/mo | 42s |
| Visakha Skin Clinic | 55 | No Google Business Profile claimed | 4 hrs/mo saved, 10–20 additional bookings/mo | 22s |

Both went from raw discovery through all five sub-agents to a Gmail draft, with zero manual intervention and zero failed executions. Sample drafted output (Skinshine):

> **Subject:** Claim Skinshine's Google Business Profile
>
> Hi there, Skinshine Skin & Hair Clinic, I've looked at your site.
>
> Your clinic's site loads in 3.5 seconds and currently has no Google Business Profile.
>
> Missing a Google Business Profile means lost local search visibility and potential clients.
>
> - Claim and optimize a Google Business Profile to boost local search rankings and attract new patients.
> - Save 7 hours weekly and expect 1–5 additional booked appointments per month.
>
> Worth a quick 20-minute free automation audit call this week?
>
> Surya, True Digital

## Reliability

- **Existence-check-before-write** - every business is checked against HubSpot by domain before any company/deal is created, preventing duplicate CRM records across repeated runs.
- **Explicit LLM fallback** - n8n's Information Extractor node doesn't support native multi-model failover despite its type signature suggesting otherwise (confirmed via `DUPLICATE_SUBNODE_CONNECTION` at runtime). Fallback is instead built explicitly: primary node set to `continueErrorOutput`, wired to a duplicate node bound to the fallback model on the error branch.
- **Draft-only outreach** - the Gmail node has draft scope only. There is no code path in the workflow that can send a message; approval is architectural, not a prompt instruction.
- **Human review, not just a batch summary** - every opportunity gets its own Slack notification (business name, confidence score, pain point, ROI, next action) rather than a single end-of-batch count, so review stays under 3 minutes per item.

## Roadmap

- Real SEO/GBP/LinkedIn API sourcing - currently LLM-estimated from scraped site content rather than pulled from SEMrush/Ahrefs/Google Business Profile APIs.
- Persistent Memory / Learning History store (Supabase) - no calibration feedback loop from outreach outcomes back into scoring yet.

## Setup

1. Import the two orchestrator workflows and five sub-agent workflows into n8n.
2. Configure credentials: Firecrawl API key, Groq API key, OpenRouter API key, HubSpot private app token, Gmail OAuth2 (draft scope only), Slack OAuth2.
3. Edit the target segment list in the `Select Target Segment` code node (service + city + state combos) to match your ICP.
4. Set the Schedule Trigger cadence, or run manually for an ad hoc batch.

## Demo

*(video walkthrough - drag the recorded MP4 directly into this file in the GitHub web editor; it auto-hosts and inserts a working embed link, no external hosting needed)*

---

*Built by Surya, True Digital.*
