# SearchUnify AI Agent QA Automation Suite

An end-to-end automation and evaluation suite for testing the **SearchUnify "SU on the Side" (AI Agent Partner)** Chrome extension against live Salesforce Lightning support cases.

The system scrapes real Salesforce cases, reads what the SU extension's AI generated for each case, and uses OpenAI to score the relevancy, accuracy, and quality of the AI output across five distinct evaluation dimensions.

---

## What This Does

When a support agent opens a Salesforce case, the SearchUnify Chrome extension provides AI-generated assistance:
- **Opening Assessment** — problem statement, root cause hypothesis, what's been tried, next steps
- **Draft Response** — a suggested reply to the customer
- **Case Summary** — a concise summary of the case
- **Customer Sentiment** — how the customer is feeling throughout the conversation
- **Case Timeline** — chronological breakdown of events

This tool automatically:
1. Connects to your already-running Chrome instance (where you're logged into Salesforce)
2. Scrapes the full case — subject, fields, feed of comments (including lazy-loaded and collapsed posts)
3. Reads what the SU extension's AI actually generated for each of the five outputs
4. Scores each output using OpenAI GPT — checking relevancy, accuracy, hallucination, tone, and completeness
5. Renders a detailed scored HTML report

---

## Screenshots

### Step 2 — Select Case
Browse pre-created test cases or paste any Salesforce Lightning case URL.

![Step 2 — Select Case (full grid)](Screenshot%20from%202026-07-15%2011-14-20.png)

*Quick Load grid showing 12 pre-seeded test scenarios with tags and comment counts.*

---

### Step 2 — Case Fetched (API Edition)
After clicking **Fetch Case**, the case details are shown inline before analysis begins.

![Case fetched — detail preview](Screenshot%20from%202026-07-15%2011-17-28.png)

*Case #00001205 loaded — shows subject, status, priority, account, contact, comment count, and full description.*

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
├── backend/
│   ├── main.py           # FastAPI app, all API endpoints
│   ├── analyzer.py       # OpenAI scoring engine, all 5 evaluation dimensions
│   ├── scraper.py        # Playwright/CDP Salesforce scraper
│   └── requirements.txt
│
├── frontend/
│   └── src/
│       ├── app/
│       │   └── page.tsx          # Main page, Step state machine
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
├── create_case_with_comments.py  # Single test case creator
├── create_18_cases.py            # 18 edge-scenario cases
├── create_sentiment_cases.py     # 6 multi-sentiment-arc cases
├── run_4case_full_report.py      # Batch pipeline for 10 cases
├── run_sentiment_cases_report.py # Batch pipeline for 6 sentiment cases
├── su_extension_helper.py        # SU extension CDP interaction helper
├── su_api_client.py              # Direct SearchUnify API client
└── .env                          # OPENAI_API_KEY (not committed)
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

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `OPENAI_API_KEY` | Yes | Your OpenAI API key |
| `OPENAI_MODEL` | No | Model to use (default: `gpt-5-mini`) |

Create a `.env` file in the **repo root** (not inside `backend/`). The backend loads it from one level up.

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

- Chrome must be pre-running with CDP flags — the app cannot start Chrome from scratch without the correct flags
- Case scraping takes ~90 seconds due to Salesforce Lightning's lazy-loading feed
- The Playwright executor uses `max_workers=1` — Playwright calls cannot be parallelized
- Case 1202 (one of the 10 batch cases) intermittently times out due to SU API latency on the draft response step

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 14, React 18, Tailwind CSS (dark theme) |
| Backend | FastAPI, Python 3.11, Uvicorn |
| Browser Automation | Playwright (sync API, CDP mode) |
| AI Scoring | OpenAI GPT (configurable model) |
| Salesforce Integration | REST API + Chatter API + CDP DOM scraping |


---

## Source Code

The complete source code for this project is not publicly available as it contains company-specific and proprietary implementation details.

For interview or verification purposes, a sanitized version of the project will be shared through the following Google Drive link:

**Google Drive:** https://drive.google.com/file/d/1rnhCg1n9skXIoWnIEf9nTvqMf5tnLUVH/view?usp=sharing

> *The Drive link will be made available after the interview process or upon request for code verification.*
