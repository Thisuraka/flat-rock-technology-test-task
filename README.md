# AI Sales Email Triage & Auto-Response Agent

An agentic AI system that classifies inbound sales emails, extracts structured data, drafts knowledge-grounded responses via RAG, and sends replies after human approval — built entirely in n8n.

`n8n` · `GPT-4o` · `Qdrant` · `LangChain` · `Gmail API` · `Slack`

> Built as a test task submission for the Agentic AI Engineer role at Flat Rock Technology.

---

## The Problem

A tech consulting firm receives dozens of sales emails daily — RFPs, vague inquiries, spam, job applications, even prompt injection attempts. A human sales team wastes hours reading every email, looking up relevant case studies, drafting replies, and logging leads into a CRM. Most of this work follows a repeatable pattern that can be automated, but the stakes are high: a bad reply loses a deal, and an unsupervised AI risks sending hallucinated or unsafe content.

## What This System Does

- **Polls** a Gmail inbox every minute for new emails (including attachments)
- **Classifies** each email into one of four categories: inquiry, follow-up, spam, or unrelated
- **Extracts attachments** — PDF text extraction and CSV-to-JSON parsing
- **Pulls structured data** from the email and attachments: sender, company, industry, project type, requirements, and a confidence score
- **Routes low-confidence extractions** (< 0.7) to Slack for manual review instead of auto-processing
- **Runs pre-RAG safety guardrails** — jailbreak detection, NSFW filter, and topical alignment checks — before the AI agent touches the email
- **Drafts a reply** using an AI Agent that performs 3+ knowledge base searches in Qdrant before writing, citing real case studies with verified metrics
- **Runs post-RAG safety guardrails** on the generated draft
- **Sends the draft to Slack** for dual human approval before any email leaves the system
- **Formats and sends** the approved draft as branded HTML via Gmail
- **Logs every email** to Google Sheets regardless of outcome — inquiries and non-inquiries alike

## Design Decisions

| Decision | Rationale |
|---|---|
| **gpt-4o-mini for classification & guardrails, gpt-4o for drafting** | Classification and safety checks are high-volume, low-complexity tasks where mini's cost-efficiency matters. Drafting requires nuance, tone-matching, and accurate citation. |
| **Confidence threshold ≥ 0.7** | Emails with ambiguous or incomplete data are flagged for manual review on Slack rather than risking a poor auto-response. |
| **Mandatory 3+ KB searches before drafting** | Prevents the AI agent from hallucinating. Every reply must be grounded in actual knowledge base content — case studies, pricing, service catalogs. |
| **Dual approval on Slack** | Human reviewers see the original email and the AI draft side by side before any reply is sent. |
| **Pre-RAG and Post-RAG guardrails** | Pre-RAG catches malicious input before the expensive agent call. Post-RAG catches unsafe output before it reaches a human reviewer. |
| **All emails logged regardless of outcome** | Spam and job applications are logged to a Non-Inquiry sheet. Nothing disappears silently. |

## Safety & Guardrails

The system implements defense-in-depth with checks at multiple pipeline stages:

- **Pre-RAG Jailbreak Detection** (threshold 0.7) — catches prompt injection attempts where attackers embed instructions to leak system prompts or confidential data
- **Pre-RAG NSFW Filter** (threshold 0.7) — blocks inappropriate content before it reaches the AI agent
- **Pre-RAG Topical Alignment** (threshold 0.7) — verifies the email relates to ABC Company's services and rejects off-topic content like job applications or unrelated solicitations
- **Post-RAG NSFW Filter** — validates the AI-generated draft itself is clean before human review
- **Agent-level constraints** — the system prompt prohibits placeholder text (`[Name]`), fabricated metrics, markdown formatting, and references to internal KB filenames
- **Human-in-the-loop** — dual Slack approval acts as the final safety net

## Knowledge Base

The AI agent searches a Qdrant vector store containing 10 documents about ABC Company, embedded using OpenAI's `text-embedding-3-small` model:

| Document | Content |
|---|---|
| Company Overview | Mission, 60-person team, offices in Colombo / London / Melbourne, ISO 27001, AWS Advanced Partner |
| Services Catalog | Custom web & mobile apps, cloud infrastructure, AI/ML, UI/UX design, QA, staff augmentation |
| Case Studies | 8 client projects across fintech, healthcare, e-commerce, SaaS, logistics, edtech, proptech, enterprise |
| Technology Stack | React, Node.js, Python, AWS, GCP, Kubernetes, and more |
| Pricing Guide | Engagement pricing by scope and model |
| Engagement Models | Project-based, dedicated team, staff augmentation |
| Development Process | Discovery through deployment methodology |
| Team Capabilities | Team structure, certifications, and expertise areas |
| FAQ | Common client questions and answers |
| Terms & Policies | Legal, NDA, and compliance information |

The agent can cite these case studies with verified metrics: **NovaPay** (fintech), **MediConnect** (healthcare), **TradeNest** (e-commerce), **InsightFlow** (SaaS), **SwiftRoute** (logistics), **LearnSpark** (edtech), **DwellDesk** (proptech), **GlobalMed Supplies** (enterprise/SAP).

## Test Scenarios

| # | Sender | Type | Attachment | What It Tests |
|---|---|---|---|---|
| 1 | Dr. Amanda Roberts, HealthFirst Inc. | Healthcare RFP | PDF (patient portal requirements) | Full happy path — classify, extract PDF, match MediConnect case study, draft RFP response, Slack approval, Gmail reply |
| 2 | Amara Okafor, LuxeMarket | E-commerce inquiry | CSV (16 marketplace features) | CSV parsing, budget assessment ($80–120K), technical recommendation, TradeNest case study citation |
| 3 | Thomas (no company) | Minimal-detail inquiry | PDF (project notes) | Low-information handling — vague body, industry detection from attachment, appropriate response despite sparse input |
| 4 | Brad Thompson, SEOBoost Pro | Spam / marketing pitch | None | Correct spam classification, no AI agent invoked, logged to Non-Inquiry sheet |
| 5 | Priya Sharma | Job application | PDF (resume) | Correct classification as unrelated, no auto-reply, logged to Non-Inquiry sheet |
| 6 | "Alex Chen, NexaPay Holdings" | Prompt injection attack | None | Jailbreak detection — email embeds instructions to leak system prompts and KB data; pre-RAG guardrails block before AI agent |

## Tech Stack

| Role | Technology |
|---|---|
| Orchestration | n8n |
| LLM — Classification & Guardrails | OpenAI GPT-4o-mini |
| LLM — Reply Drafting | OpenAI GPT-4o |
| Embeddings | OpenAI text-embedding-3-small |
| Vector Database | Qdrant |
| Safety | LangChain Guardrails (jailbreak, NSFW, topical) |
| Email I/O | Gmail API (OAuth2) |
| Human Approval | Slack (dual approval flow) |
| CRM Logging | Google Sheets |
| Structured Extraction | LangChain Information Extractor |
| Output Parsing | LangChain Structured Output Parser |

## Project Structure

```
flat-rock-tech/
  Task.json                     # Complete n8n workflow (importable)
  knowledge_base/               # 10 ABC Company PDFs for Qdrant
    abc_01_company_overview.pdf
    abc_02_services_catalog.pdf
    ...
    abc_10_terms_policies.pdf
  test_emails/                  # 6 test scenarios with attachments
    email_1_healthfirst/
    email_2_luxemarket/
    email_3_logistics/
    email_4_seoboost/
    email_5_jobapplication/
    email_6_jailbreak/
```
