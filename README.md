# SearchUnify AI Agent QA & Relevancy Evaluation Suite

An **LLM evaluation and QA platform** for the **SearchUnify "SU on the Side" (AI Agent Partner)** — a generative AI copilot embedded in Salesforce support workflows. This repo ships **two complementary applications** that answer the same question — *"is the AI actually right?"* — from two different angles:

| | [`backend/` + `frontend/`](#-app-1-functional-testing-suite-browser-automation) | [`api_app/`](#-app-2-relevancy-analysis-engine-api-based) |
|---|---|---|
| **Purpose** | End-to-end functional/UI testing of the live Chrome extension | Automated relevancy & hallucination evaluation at scale |
| **Data source** | Browser automation (Playwright + Chrome DevTools Protocol) | Direct REST/SSE API integration (SearchUnify + Salesforce) |
| **Ground truth** | Scraped Salesforce Lightning DOM | Salesforce SOQL + OAuth2, queried directly |
| **Best for** | Regression-testing the extension's actual UI/UX end-to-end | CI-friendly, headless, scalable LLM-output grading |
| **Core technique** | CDP-driven DOM scraping & UI interaction | **LLM-as-a-Judge** + **claim decomposition/verification** (hallucination-resistant scoring) |

Both apps share one evaluation core (`backend/analyzer.py`) — a from-scratch **grounded LLM evaluation engine** built for exactly the kind of problem generative AI teams are hiring for right now: *scoring open-ended LLM output against ground truth, deterministically, with evidence, at scale.*

---

## Source Code

The complete source code for this project is not publicly available as it contains company-specific and proprietary implementation details.

For interview or verification purposes, a sanitized version of the project will be shared through the following Google Drive link:

**Google Drive:** [Drive Link](https://drive.google.com/file/d/1rnhCg1n9skXIoWnIEf9nTvqMf5tnLUVH/view?usp=sharing)

> *The Drive link will be made available after the interview process or upon request for code verification.*

## What This Does

When a support agent opens a Salesforce case, the SearchUnify AI Agent Partner generates several pieces of AI output, each independently evaluated:
- **Opening Assessment** — problem statement, root cause hypothesis, what's been tried, next steps
- **Draft Response** — a suggested reply to the customer
- **Case Summary** — a concise summary of the case
- **Customer Sentiment** — how the customer is feeling throughout the conversation
- **Case Timeline** — chronological breakdown of events
- **Account Health** *(`api_app` only)* — RAG (red/amber/green) status derived from account-level Salesforce metrics
- **Escalation Risk** *(`api_app` only)* — a weighted 0–100% risk band combining frustration signal, SLA breach data, and LLM-judged case severity/complexity

Both apps:
1. Retrieve the ground-truth case record (via browser scrape or direct API/SOQL)
2. Retrieve what the AI actually generated for each dimension
3. Score each output with an LLM evaluation pipeline — checking relevancy, factual accuracy, hallucination, tone, and completeness
4. Render scored, exportable reports (HTML/CSV)

---

# 🧭 App 1: Functional Testing Suite (Browser Automation)

**Location:** [`backend/`](backend/) + [`frontend/`](frontend/)

Drives the real Chrome extension via CDP, exactly as a support agent would experience it — the gold standard for catching UI regressions, broken selectors, and behavior that only shows up in the actual rendered product.

## Screenshots

### Step 2 — Select Case
Browse pre-created test cases or paste any Salesforce Lightning case URL.

![Step 2 — Select Case (full grid)](Screenshot%20from%202026-07-15%2011-14-20.png)

*Quick Load grid showing 12 pre-seeded test scenarios with tags and comment counts.*

---

### Step 3 — Analysis Results (Opening Assessment)
The scored breakdown for the Opening Assessment across all 7 sections.

![Analysis results — Opening Assessment](Screenshot%20from%202026-07-15%2011-21-35.png)

*Case #00001205 scored 8.6 (Strong / PASS). Each section shows weighted score, verdict, and reasoning. Export buttons on the right generate HTML or CSV reports.*

---

### Batch Report — Multi-Case Analysis
The generated HTML report covering multiple cases at once.

![Multi-Case Analysis Report](Screenshot%20from%202026-07-15%2011-12-31.png)

*5-case report: overall score 8.1/10, 4 PASS, 1 PARTIAL. Each case is expandable with full dimension breakdown.*

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                   Next.js Frontend (port 3001)          │
│  AuthStep → CaseStep → AnalysisStep / BatchAnalysisStep │
└────────────────────┬────────────────────────────────────┘
                     │ HTTP (direct, 150–240s timeout)
┌────────────────────▼────────────────────────────────────┐
│              FastAPI Backend (port 8000)                 │
│  ┌──────────────────┐   ┌──────────────────────────────┐│
│  │   scraper.py     │   │        analyzer.py           ││
│  │  Playwright+CDP  │   │  OpenAI GPT scoring engine   ││
│  │  Salesforce DOM  │   │  5 evaluation dimensions     ││
│  └────────┬─────────┘   └──────────────────────────────┘│
└───────────┼─────────────────────────────────────────────┘
            │ CDP WebSocket (port 9222)
┌───────────▼──────────────────────────────────────────────┐
│         Chrome (already running, user-managed)            │
│   Salesforce Lightning  +  SU Extension iframe (CDN)      │
└──────────────────────────────────────────────────────────┘
```

The backend has **two separate thread pools**:
- `_playwright_executor` — `max_workers=1`, all Playwright calls must run on this single thread to avoid greenlet switching errors
- `_openai_executor` — `max_workers=4`, for parallel OpenAI/CPU work

---

## Prerequisites

| Requirement | Details |
|---|---|
| Python 3.11+ | Backend runtime |
| Node.js 18+ | Frontend runtime |
| OpenAI API Key | GPT model access |
| Google Chrome | Must be running with CDP enabled (see below) |
| Salesforce access | Logged in via the CDP Chrome profile |
| SU Extension | SearchUnify "SU on the Side" Chrome extension installed |

---

## Chrome Setup (Required — Do This First)

Everything connects to **one already-running Chrome instance** over the Chrome DevTools Protocol (CDP). Chrome must be started with these flags:

```bash
google-chrome \
  --remote-debugging-port=9222 \
  --user-data-dir=~/.config/chrome-selenium-debug \
  --remote-allow-origins=* \
  --no-first-run \
  --no-default-browser-check
```

> **`--remote-allow-origins=*` is mandatory.** Without it, the SU extension iframe's CDP target rejects WebSocket connections.

Once Chrome is running:
1. Log into your **Salesforce org** in that Chrome window
2. Log into the **Community portal** (for customer-side comment posting)
3. The SU extension should be active and visible in the sidebar when you open a case

---

## Installation

### 1. Clone and configure environment

```bash
git clone <this-repo>
cd AH-automation-suse

# Create .env in the repo root
echo "OPENAI_API_KEY=sk-..." > .env
echo "OPENAI_MODEL=gpt-4o-mini" >> .env   # optional, default is gpt-5-mini
```

### 2. Backend setup

```bash
cd backend
pip install -r requirements.txt
playwright install chromium
```

### 3. Frontend setup

```bash
cd frontend
npm install
```

---

## Running the Web App

Start both servers (in separate terminals):

```bash
# Terminal 1 — Backend
cd backend
uvicorn main:app --reload --port 8000 --timeout-keep-alive 300

# Terminal 2 — Frontend
cd frontend
npm run dev          # runs on port 3001
```

Open [http://localhost:3001](http://localhost:3001) in your browser.

> The frontend calls the backend **directly** at `http://localhost:8000/api` (not through the Next.js proxy) to avoid the proxy's shorter timeout. Long scrape and analysis operations take 90–240 seconds.

---

## Using the Web App

The UI is a 3-step flow:

### Step 1 — Auth
Verifies Chrome is running on port 9222 and that you are logged into Salesforce. If Chrome is not running, a "Start Chrome" button launches it.

### Step 2 — Select Case
Enter a Salesforce case URL or pick from the pre-seeded quick-select list (`frontend/src/lib/testCases.ts`). The scraper fetches the full case including all feed comments (~90 seconds).

### Step 3 — Analysis
Runs all five evaluation dimensions. Each dimension:
1. Reads the SU extension's actual output (via CDP into the extension iframe)
2. Scores it against the scraped case context using GPT
3. Displays scored results with section-level breakdown

You can also use **Batch Analysis** to run multiple cases in sequence.

---

## Five Evaluation Dimensions

### 1. Opening Assessment
Scores 7 fixed sections of the AI's case assessment:

| Section | What's scored |
|---|---|
| Problem Statement | PASS / PARTIAL / FAIL / NOT_APPLICABLE |
| Classification | PASS / PARTIAL / FAIL / NOT_APPLICABLE |
| Root Cause Hypothesis | PASS / PARTIAL / FAIL / NOT_APPLICABLE |
| Related Findings | PASS / PARTIAL / FAIL / NOT_APPLICABLE |
| What's Been Tried | PASS / PARTIAL / FAIL / NOT_APPLICABLE |
| What's Been Recommended | PASS / PARTIAL / FAIL / NOT_APPLICABLE |
| Next Best Actions | PASS / PARTIAL / FAIL / NOT_APPLICABLE |

NOT_APPLICABLE sections are excluded from the overall score.

### 2. Draft Response
Scores the AI's suggested reply on: relevancy, accuracy, tone, completeness, case alignment.

### 3. Case Summary
Scores: relevancy, accuracy, completeness, current state reflection, consistency with case feed.

### 4. Customer Sentiment
Evaluates how accurately the extension detected the customer's sentiment.  
Allowed labels: **Positive | Neutral | Negative | Frustrated**

### 5. Sentiment × Timeline (Combined)
The most sophisticated evaluation — a 3-way comparison:
- **SU Reported** — what the extension labeled the sentiment
- **Derived (Numeric Average)** — computed from per-entry API sentiment scores
- **LLM Independent Judgment** — GPT reads all customer messages and assigns a label independently

All three are compared and discrepancies are flagged.

---

## Batch Reporting Scripts

For bulk analysis without the UI:

| Script | Purpose |
|---|---|
| `run_4case_full_report.py` | Runs all 5 dimensions on 10 pre-seeded cases, saves `4case_full_cache.json`, generates `full_analysis_report.html` |
| `run_sentiment_cases_report.py` | Same pipeline for the 6 sentiment-arc test cases |
| `patch_sentiment_3cases.py` | Re-runs only the sentiment×timeline step on specific cached cases |

Cache files (`*_cache.json`) are whole-case level — delete a case entry and re-run to force a fresh analysis for that case.

---

## Test Case Creation Scripts

Scripts for seeding test data into Salesforce:

| Script | What it creates |
|---|---|
| `create_case_with_comments.py` | Single case with 20 realistic agent/customer comments |
| `create_18_cases.py` | 18 cases covering edge scenarios (empty feed, reopened cases, multi-issue, angry customer, sarcasm, multilingual, etc.) |
| `create_sentiment_cases.py` | 6 cases with intentional multi-sentiment arcs for testing the sentiment×timeline evaluation |

All creation scripts:
- Connect to the running Chrome instance to extract the Salesforce session cookie
- Create cases via the Salesforce REST API
- Post **agent comments** via the Chatter API (internal and public visibility)
- Post **customer comments** via the Community portal using Playwright (so they appear as community user messages)
- Are idempotent — re-running skips already-created cases

### The 6 Sentiment-Arc Test Cases

Specifically designed to challenge the sentiment analysis:

| # | Arc | Scenario |
|---|-----|----------|
| 31 | Neutral → Panicked → Relieved → Grateful | All content disappeared after maintenance |
| 32 | Excited → Frustrated → Resigned → Cautiously Satisfied | SSO blocking 500 users on launch day |
| 33 | Anxious → Near-Panic → Reassured → Satisfied | GDPR/data residency compliance concern |
| 34 | Aggressively Frustrated → Brief Relief → Angry → Grateful | AI bot quoting wrong pricing (3rd recurrence) |
| 35 | Neutral/Professional → Skeptical → Frustrated → Grateful | 4.8s latency vs 500ms SLA |
| 36 | Very Positive → Concerned → Frustrated → Distraught → Grateful | Premium features locked after $34k renewal |

---

## Project Structure

```
AH-automation-suse/
├── backend/                      # ── App 1: browser-automation functional test suite ──
│   ├── main.py                   # FastAPI app, all API endpoints
│   ├── analyzer.py               # Shared evaluation engine — LLM-as-judge + claim
│   │                             # decomposition/verification, all 7 dimensions
│   │                             # (imported directly by App 2 via importlib)
│   ├── scraper.py                # Playwright/CDP Salesforce DOM scraper
│   └── requirements.txt
│
├── frontend/                     # ── App 1 UI (Next.js, port 3001) ──
│   └── src/
│       ├── app/page.tsx          # Main page, Step state machine
│       ├── components/
│       │   ├── AuthStep.tsx      # Chrome + Salesforce connection
│       │   ├── CaseStep.tsx      # Case URL input + quick-select
│       │   ├── AnalysisStep.tsx  # Runs + renders all 5 dimensions
│       │   └── BatchAnalysisStep.tsx
│       └── lib/
│           ├── api.ts            # Backend API client + TypeScript types
│           ├── testCases.ts      # Pre-seeded case IDs for quick-select
│           └── exportReport.ts   # HTML report export
│
├── api_app/                       # ── App 2: API-based relevancy analysis engine ──
│   ├── backend/
│   │   ├── main.py               # FastAPI app (port 8001), 7 evaluation endpoints
│   │   ├── su_client.py          # SearchUnify AI Agent Partner REST/SSE client
│   │   ├── sf_client.py          # Salesforce OAuth2 + SOQL ground-truth client
│   │   ├── analyzer.py           # Thin re-export shim → backend/analyzer.py
│   │   ├── langfuse_tracer.py    # LLM observability — traces every OpenAI call
│   │   └── requirements.txt
│   └── frontend/
│       └── src/
│           ├── app/page.tsx      # connecting → home → analysis state machine
│           └── components/
│               ├── CaseStep.tsx      # Case lookup by number
│               ├── AnalysisStep.tsx  # Renders all 7 dimensions, method toggle
│               └── BatchStep.tsx     # Sequential multi-case batch analysis
│
├── create_case_with_comments.py  # Single test case creator
├── create_18_cases.py            # 18 edge-scenario cases
├── create_sentiment_cases.py     # 6 multi-sentiment-arc cases
├── run_4case_full_report.py      # Batch pipeline for 10 cases
├── run_sentiment_cases_report.py # Batch pipeline for 6 sentiment cases
├── su_extension_helper.py        # SU extension CDP interaction helper (App 1)
├── su_api_client.py              # Standalone direct SearchUnify API client (scripts)
└── .env                          # API keys / OAuth secrets (not committed)
```

---

## API Endpoints

All endpoints are under `/api/`. The full list:

| Method | Endpoint | Description |
|---|---|---|
| GET | `/health` | Liveness check |
| GET | `/auth/status` | Chrome running + Salesforce login state |
| POST | `/auth/start-chrome` | Launches Chrome with CDP flags |
| POST | `/case/fetch` | Full ~90s case scrape |
| POST | `/extension/read` | Reads Opening Assessment raw text from extension |
| POST | `/analysis/run` | Scores Opening Assessment |
| GET | `/extension/draft-response` | Clicks button, reads draft text |
| POST | `/analysis/draft-response` | Reads + scores Draft Response |
| GET | `/extension/case-summary` | Clicks button, reads summary |
| POST | `/analysis/case-summary` | Reads + scores Case Summary |
| POST | `/analysis/customer-sentiment` | Reads + scores Customer Sentiment |
| POST | `/analysis/case-timeline` | Reads + scores Case Timeline |

---

## Hard-Won DOM Facts (Salesforce Scraping)

The Salesforce Lightning feed has several non-obvious behaviours documented in `scraper.py`:

- **Top-level posts**: `article.cuf-feedElement` — **Nested replies**: `article.cuf-commentItem`
- The feed **only lazy-loads on a real mouse-wheel scroll** — `scrollTop` / `window.scrollTo` does not trigger it
- Collapsed posts lack `div.cuf-feedBodyText` and must be clicked to expand before reading body text
- The SU extension's real UI lives inside `<iframe class="shell-iframe">` pointing at a CloudFront URL — a **separate CDP target** from the main page
- A full case scrape takes ~90 seconds; all client timeouts are sized accordingly

---

# 🧠 App 2: Relevancy Analysis Engine (API-Based)

**Location:** [`api_app/backend/`](api_app/backend/) + [`api_app/frontend/`](api_app/frontend/)

Zero browser, zero DOM scraping. This app talks **directly to the SearchUnify and Salesforce APIs**, making it fast, headless, and CI-friendly — the shape you'd want for continuously grading a production LLM feature at scale rather than clicking through a UI.

It answers the harder question in applied GenAI right now: *given open-ended, unstructured LLM output, how do you score factual correctness against ground truth — automatically, defensibly, and without an LLM just grading its own homework?*

![Case fetched — API-based detail preview](Screenshots/Screenshot%20from%202026-07-15%2011-17-28.png)

*Case #00001205 fetched directly via the Salesforce API — subject, status, priority, account, contact, comment count, and full description, no browser involved.*

### Data integrations

| Integration | Client | Details |
|---|---|---|
| **SearchUnify AI Agent Partner API** | `su_client.py` | REST + **Server-Sent Events (SSE)** streaming against `POST /v1/sse/ai-agent-partner`. Bearer-token auth (`X-CRM-Platform`, `X-Instance-Url`, `X-UUX-App-Id`, per-request `X-Request-Id`). One streaming action (`greeting`) drives the Opening Assessment; a `chat` action with dimension-specific system prompts drives Draft Response, Case Summary, Timeline, Sentiment, Account Health, and Escalation Risk. |
| **Salesforce** | `sf_client.py` | OAuth2 **client-credentials** flow (token cached, auto-refreshed on `401`) + **SOQL** against `Case`, `FeedItem`, `FeedComment`, `CaseComment`, `EmailMessage`, and `CaseMilestone` — merged into the same chronological case-record shape the browser scraper produces, so both apps share one evaluation contract. |

### The Evaluation Engine — grounded LLM-as-a-Judge

The scoring core (shared with App 1, via `backend/analyzer.py`) exposes **two selectable evaluation methods** per request, controlled by a `method` parameter (`"llm"` | `"claim"`):

**1. `"llm"` — Single-call LLM-as-a-Judge**
One structured, JSON-mode chat completion scores the output directly against the case context — fast, simple, the industry-standard baseline for LLM evaluation pipelines.

**2. `"claim"` (default) — Decompose-Verify: a hallucination-resistant, evidence-grounded pipeline**

Rather than trusting a single LLM verdict, this runs a 4-stage **claim decomposition & verification** technique — the same family of methods used in modern factuality-benchmarking research (e.g. FActScore-style atomic fact-checking):

1. **Extract** — the AI's output is decomposed into atomic `factual_assertion` vs. `judgment` claims (temperature `0.0` for determinism).
2. **Verify** — every factual claim is independently checked against the case record and labeled `SUPPORTED` / `CONTRADICTED` / `NOT_FOUND`, each requiring a **quoted evidence span** — claims are shuffled first to reduce positional/order bias.
3. **Keyfacts & Coverage** — a second, *assessment-blind* pass extracts the case's key facts independently, then measures whether the AI's output actually covered them (a recall/completeness metric, decoupled from the hallucination check).
4. **Reverify** — a second-pass judge re-examines every flagged claim, requiring a quotable span before it counts as a confirmed `HALLUCINATION` — this materially cuts false-positive hallucination flags and reclassifies weak flags as `RECOVERED`.

The final score is **arithmetic, not LLM-emitted** — every point is traceable to a specific claim and its supporting evidence:

```
score = 10 × faithfulness × (0.5 + 0.5 × coverage) − contradiction_penalty
```

This is the difference between "an LLM said it's an 8/10" and "here are the 11 claims we checked, the 2 we couldn't verify, and the evidence for each" — auditable, explainable AI evaluation rather than a black-box score.

### Structured output & reliability engineering

- **JSON-mode structured outputs** (`response_format={"type": "json_object"}`) on every OpenAI call — no brittle regex/markdown parsing of LLM responses.
- Model-aware **temperature handling** — reasoning models (GPT-5 / o-series) reject non-default temperature, so it's conditionally omitted per model family.
- **Ground-truth-then-judge pattern** for the two most numeric dimensions:
  - **Account Health** — a deterministic RAG (red/amber/green) status is computed *first* from real Salesforce metrics (open case count, critical/high severity count, SLA-violation count, 90-day escalation rate), and the LLM is then graded against that verified baseline — not against its own opinion.
  - **Escalation Risk** — combines a pre-computed frustration score and an SLA-breach score with an LLM-judged severity/complexity rating into a weighted 0–100% risk band (Low/Medium/High), including a "resolution dampener" rule that only trusts the customer's *latest* message when deciding if a risk has actually been resolved.
- **Full observability** — `langfuse_tracer.py` transparently instruments every OpenAI call as a nested Langfuse **trace → span → generation**, using `contextvars` so concurrent async requests each track their own trace safely. Full prompt/response/token-usage visibility with zero changes to the evaluation logic itself — a production-grade LLM-ops observability pattern.

### Evaluation dimensions (7 total)

| Dimension | What's measured |
|---|---|
| Opening Assessment | 7 weighted sections (problem statement, classification, root cause, etc.) |
| Draft Response | Relevancy, accuracy, tone, completeness, case alignment |
| Case Summary | Relevancy, accuracy, completeness, current-state accuracy, consistency |
| Customer Sentiment | Sentiment-label accuracy vs. case tone |
| Case Timeline | Chronological accuracy and completeness |
| **Account Health** | RAG status vs. deterministic Salesforce-derived ground truth |
| **Escalation Risk** | Weighted risk-band accuracy, reasoning quality, coverage, hallucination check |

### App 2 API Endpoints

All endpoints run on port `8001` under `/api/`. Every analysis endpoint accepts `model` (default `gpt-5-mini`) and `method` (`"llm"` | `"claim"`) in the request body.

| Method | Endpoint | Description |
|---|---|---|
| GET | `/health` | Liveness check |
| GET | `/connection/check` | Verifies Salesforce OAuth token + SearchUnify API reachability |
| POST | `/case/fetch` | Fetches case record directly via Salesforce SOQL (no browser) |
| POST | `/analysis/run` | Scores Opening Assessment |
| POST | `/analysis/draft-response` | Scores Draft Response |
| POST | `/analysis/case-summary` | Scores Case Summary |
| POST | `/analysis/customer-sentiment` | Scores Customer Sentiment |
| POST | `/analysis/case-timeline` | Scores Case Timeline |
| POST | `/analysis/account-health` | RAG-status ground truth vs. LLM evaluation |
| POST | `/analysis/escalation-risk` | Weighted risk-band scoring vs. LLM evaluation |

### Running App 2

```bash
cd api_app/backend
pip install -r requirements.txt
uvicorn main:app --reload --port 8001

cd api_app/frontend
npm install
npm run dev
```

Configure `SU_ENDPOINT`, `SU_OPAQUE_TOKEN`, Salesforce OAuth2 client credentials, and (optionally) `LANGFUSE_*` keys — see [Environment Variables](#environment-variables).

The frontend flow (`connecting → home → analysis`) lets you toggle the evaluation `method` (**LLM Judge** vs. **Decompose + Verify**) per run, supports single-case and **batch** analysis, and warns if cached results were scored with a different method than currently selected.

---

## Environment Variables

**Shared (both apps)**

| Variable | Required | Description |
|---|---|---|
| `OPENAI_API_KEY` | Yes | Your OpenAI API key |
| `OPENAI_MODEL` | No | Model to use (default: `gpt-5-mini`) |

**App 2 only (`api_app/`) — API-based integrations**

| Variable | Required | Description |
|---|---|---|
| `SU_ENDPOINT` | No | SearchUnify AI Agent Partner API base URL (default: `https://feature4.searchunify.com/unify`) |
| `SU_OPAQUE_TOKEN` | Yes | Bearer token for SearchUnify API auth |
| `SU_UID` | Yes | SearchUnify user/agent identifier |
| `SU_AGENT_NAME` | No | Agent display name sent to the API (default: `Kanav`) |
| `SF_CLIENT_ID` / `SF_CLIENT_SECRET` | Yes | Salesforce Connected App OAuth2 client-credentials |
| `SF_INSTANCE_URL` | Yes | Salesforce instance base URL |
| `SF_API_VERSION` | No | Salesforce REST API version (default: `v66.0`) |
| `LF_PUBLIC_KEY` / `LF_SECRET_KEY` | No | [Langfuse](https://langfuse.com) keys — enables full LLM tracing/observability |
| `LF_HOST` | No | Langfuse host (default: `https://langfuse.searchunify.com`) |

Create a `.env` file in the **repo root** (not inside `backend/` or `api_app/backend/`). Both backends load it from one level up.

---

## Outputs

| File | Description |
|---|---|
| `full_analysis_report.html` | Main scored HTML report (10 cases) |
| `4case_full_cache.json` | Cached analysis results for all 10 cases |
| `sentiment_cases_results.json` | Case creation results for 6 sentiment cases |
| `scenario_results.json` | Case creation results for 18 edge cases |
| `*.csv` | Exported case content for offline analysis |

---

## Known Limitations

**App 1 (browser automation)**
- Chrome must be pre-running with CDP flags — the app cannot start Chrome from scratch without the correct flags
- Case scraping takes ~90 seconds due to Salesforce Lightning's lazy-loading feed
- The Playwright executor uses `max_workers=1` — Playwright calls cannot be parallelized
- Case 1202 (one of the 10 batch cases) intermittently times out due to SU API latency on the draft response step

**App 2 (API-based)**
- Batch analysis runs cases **sequentially**, not in parallel (no `Promise.all` in the frontend batch loop)
- Requires valid SearchUnify API credentials and a Salesforce Connected App configured for OAuth2 client-credentials — not usable without backend access to both systems
- The `"claim"` (Decompose-Verify) method issues several more LLM calls per evaluation than `"llm"` mode — higher latency and token cost in exchange for evidence-grounded, hallucination-resistant scoring

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 14, React 18, Tailwind CSS (dark theme) |
| Backend | FastAPI, Python 3.11, Uvicorn |
| Browser Automation *(App 1)* | Playwright (sync API, Chrome DevTools Protocol) |
| API Integration *(App 2)* | REST + Server-Sent Events (SSE) streaming, OAuth2 client-credentials |
| AI Evaluation | OpenAI GPT (configurable model) — **LLM-as-a-Judge** & **claim decomposition/verification** (grounded, hallucination-resistant scoring) |
| Structured Output | OpenAI JSON-mode (`response_format=json_object`) — schema-reliable LLM responses |
| Observability *(App 2)* | [Langfuse](https://langfuse.com) — full trace/span/generation tracking per analysis run |
| Salesforce Integration | REST API + Chatter API + CDP DOM scraping *(App 1)*; OAuth2 + SOQL *(App 2)* |
