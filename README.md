# FDA Warning Letter Intelligence Tool

An AI-powered inspection readiness platform that transforms the FDA's public warning letter corpus into actionable compliance intelligence. Ask natural language questions and receive structured regulatory briefs backed by real enforcement data.

https://github.com/user-attachments/assets/46aabc64-ab2c-4fd6-9d02-614523d9ee49

---

## What It Does

The tool answers questions like:

- *"What CGMP violations appeared most in 2024–2025 warning letters?"*
- *"Which oncology sponsors received GCP-related warning letters recently?"*
- *"How has FDA enforcement changed in the last 3 years?"*



It does this by running an AI agent that searches a local database of 3,500+ real FDA warning letters, finds the most relevant ones (already pre-classified), and synthesises a structured inspection readiness brief — typically in under 10 seconds.

---

## Architecture Overview

```
+--------------------------------------------------------+
|           Flask + HTML UI  (server.py)                 |
|           localhost:8000  (SSE streaming)              |
+-------------------+------------------------------------+
                    | user question
                    v
+----------------------------------+
|   Query Parser (no LLM)          |  ai/query_parser.py
|   regex + keyword table          |  0 ms, deterministic
+----------------+-----------------+
                 | category, keyword, year_start, year_end
                 v
+----------------------------+
|   search_letter_index      |  ai/tools.py  <- SQLite query
|   returns <= 20 letters    |                  < 5 ms, local
+-------------+--------------+
              |
              v
+----------------------------+
|     synthesize_brief       |  ai/tools.py
|   Groq 70b / Gemini flash  |  only LLM call — reads letters, writes brief
+----------------------------+
              |
              v
+----------------------------+
|     SQLite Database        |  data/warning_letters.db
|  letters + classifications |  3,513 letters, fully pre-classified
+----------------------------+
              ^
              | populated by
+-------------+--------------+
|  Data Pipeline (offline)   |
|  scraper -> fetcher ->     |
|  classifier                |
+----------------------------+
              ^
              |
       FDA.gov (public data)
```

---

## Data Pipeline

The data pipeline runs **once** (or periodically to refresh). It has three stages that feed into each other.

### Stage 1 — Index Scraper (`scraper/fda_scraper.py`)

**What it does:** Pulls the complete warning letter index from FDA.gov.

**How it works:**

FDA.gov renders its warning letter table as a Drupal DataTables widget. Instead of scraping HTML pages, the scraper calls the underlying AJAX endpoint directly:

```
POST https://www.fda.gov/datatables/views/ajax
     view_name=warning_letter_solr_index
```

This endpoint returns JSON with HTML-encoded cell values. The scraper paginates through batches of 500 records, parses each cell (dates from `<time>` tags, company name/URL from `<a>` tags, plain text for other fields), and upserts records into SQLite keyed on `letter_url`.

**Fields collected per letter:**
- Posted date, letter issue date
- Company name + FDA.gov URL
- Issuing office (e.g., CDER, CDRH, ORA)
- Subject line
- Response letter URL (if company responded)
- Closeout letter URL (if FDA closed the case)

**Run it:**
```
python -m scraper.fda_scraper               # full index scrape (~7 pages, ~1 min)
python -m scraper.fda_scraper --incremental # stop early once all-known batch hit
```

The `--incremental` flag loads all known `letter_url`s from the DB, then stops paginating as soon as a complete batch of 500 contains no new URLs. For routine refreshes this means 1–2 AJAX requests instead of 7+. Use the full (non-incremental) command for the first run or after a long gap.

**Output:** `data/warning_letters.db` — `letters` table, 3,513 rows as of June 2026.

---

### Stage 2 — Full Text Fetcher (`scraper/letter_fetcher.py`)

**What it does:** Downloads the full body text of each warning letter and caches it in the database.

**How it works:**

For each letter URL in the index that doesn't yet have `full_text`, the fetcher:
1. GETs the FDA.gov letter page
2. Finds the `<article>` element
3. Removes both `<aside>` blocks (left navigation + right metadata panel)
4. Strips scripts, styles, and boilerplate
5. Collapses whitespace into clean plain text
6. Stores in `letters.full_text`

It is resume-safe — stop it at any time and restart; already-fetched letters are skipped. Polite 1.2-second delay between requests. Exponential back-off (5s → 15s → 45s) on errors.

**Run it:**
```
python -m scraper.letter_fetcher              # all pending
python -m scraper.letter_fetcher --year 2024  # one year only
python -m scraper.letter_fetcher --limit 100  # first 100
```

**Output:** `letters.full_text` column populated. Currently: all 3,513 letters fetched.

---

### Stage 3 — AI Classifier (`ai/classifier.py`)

**What it does:** Reads each letter's full text and extracts structured violation metadata using an LLM.

**How it works:**

For each letter with `full_text` that hasn't been classified yet, the classifier:
1. Truncates the letter to 12,000 characters (most are shorter)
2. Sends it to the LLM provider pool with a structured extraction prompt
3. Parses the JSON response
4. Saves the classification to the `classifications` table

**What gets extracted:**

| Field | Description |
|---|---|
| `violation_types` | List of violation categories — 18 canonical types (see below) |
| `gcp_sections` | ICH E6(R3) or 21 CFR sections explicitly cited (e.g. "8.3.1", "21 CFR 312.62") |
| `severity_signal` | `high` / `medium` / `low` based on clinical risk indicators |
| `therapeutic_area` | Oncology, cardiology, etc. (if determinable) |
| `company_type` | `sponsor` / `cro` / `investigator_site` / `manufacturer` |
| `key_findings` | Up to 5 specific findings in plain language |
| `enforcement_context` | One sentence on what triggered the letter |

**Canonical violation types (18):**

```
Data integrity / inadequate source records
Protocol adherence / unauthorized deviations
Informed consent deficiencies
Investigational product accountability
IRB/ethics committee failures
CGMP manufacturing violations
Import/export violations
Unapproved / misbranded product
Refusal of inspection
Adverse event reporting failures
Record keeping deficiencies
Sponsor oversight failures
Site monitoring failures
Equipment / facility deficiencies
Misbranding
Adulteration
Advertising / promotional violations
Insanitary conditions
```

LLM output is normalised against this list after classification. Any non-canonical string is mapped via a 240+ entry keyword table (`_VIOL_KEYWORDS`) — no re-classification API calls needed.

**Run it:**
```
python -m ai.classifier               # all unclassified
python -m ai.classifier --year 2024   # one year
python -m ai.classifier --limit 50    # first 50
python -m ai.classifier --reclassify  # re-run all (overwrites)
python -m ai.classifier --normalize   # fix violation_types in-place (no API calls)
python -m ai.classifier --backfill    # re-extract from cached raw_json (no API calls)
python -m ai.classifier --fill-text   # fill empty violation_types from letter text (no API calls)
python -m ai.classifier --provider ollama --model llama3.2
```

Currently: all 3,513 letters classified.

---

## AI Agent

The agent (`ai/agent.py`) is the interactive intelligence layer. It runs when the user submits a question in the Query Agent tab.

### How the Agent Works

Every query follows a fixed 3-step pipeline. **Only one step uses an LLM.**

```
Step 1 — Parse (0 ms, no LLM)
  ai/query_parser.py extracts structured search parameters from the
  user's question using regex and a keyword lookup table:
    "What CGMP violations appeared in 2024?"
     -> category="CGMP manufacturing violations", year_start=2024, year_end=2024

  This is deterministic — same question always produces the same params.
  No LLM weights involved, no hallucination risk, works offline.

Step 2 — Search (< 5 ms, no LLM)
  search_letter_index runs a SQLite query using the parsed params.
  Returns up to 20 letter dicts, each already pre-classified
  (violation_types, severity_signal, therapeutic_area, company_type).
  If nothing is found, automatically retries with a plain keyword search.

Step 3 — Synthesize (~3–8 s, one LLM call)
  synthesize_brief passes all 20 letters + the original question to
  Groq 70b (or Gemini as fallback). The model reads the letter data
  and writes a structured inspection readiness brief.
  This is the only step that genuinely needs LLM intelligence.
```

**Total time: ~3–8 seconds** (synthesis only — parsing and search are instant).

The database has been fully pre-classified offline (all 3,513 letters). The agent never classifies at query time — it reads `violation_types`, `severity_signal`, and `therapeutic_area` fields already in the DB.

### Example: step-by-step trace

Query: *"What CGMP violations appeared in 2024?"*

```
0.0s  [agent] parsed: category='CGMP manufacturing violations', year=2024
        query_parser.py matched "CGMP" -> canonical violation type.
        Extracted year from "2024" with regex. No LLM involved.

0.0s  [agent] search_letter_index(category='CGMP manufacturing violations', year=2024)
        SQLite query runs instantly:
          SELECT ... FROM letters l
          JOIN classifications c ON c.letter_id = l.id
          WHERE c.violation_types LIKE '%CGMP%'
            AND year = 2024
          ORDER BY posted_date DESC LIMIT 20
        Returns 20 pre-classified letter dicts in < 5 ms.

0.8s  [agent] synthesize_brief(20 letters)
        The 20 letter dicts are passed to synthesize_brief.

0.8s  [agent] synthesis: groq_1 (llama-3.3-70b-versatile)
        A single prompt is sent to Groq 70b:
          "Here are 20 FDA warning letters (JSON).
           Answer: 'What CGMP violations appeared in 2024?'
           Write: summary, top patterns, risk level, recommendations."
        Groq 70b reads all letters and writes the structured brief.

4.9s  Answer streamed to browser.
      Source letter cards shown (company, date, severity, violation tags).
```

### Example: varied queries — what the parser extracts

```
"Show me data integrity issues from 2022 to 2024"
  -> category='Data integrity / record keeping violations', year=2022–2024

"Pfizer warning letters last 3 years"
  -> keyword='Pfizer', year=2023–2026

"Which sponsors received GCP letters recently?"
  -> category='GCP / clinical trial violations', year=2025–2026

"What happened with import alerts in 2023?"
  -> category='Import/export violations', year=2023

"Latest adverse event reporting failures"
  -> category='Adverse event reporting failures', year=2025–2026
```

### Example: when synthesis Groq key is slow

The only LLM call left is synthesis. If one key is slow, the next is tried:

```
0.0s  [agent] parsed: category='CGMP manufacturing violations', year=2024
0.0s  [agent] search_letter_index(...)   <- instant
0.5s  [agent] synthesize_brief(20 letters)
0.5s  [agent] synthesis: groq_1 (llama-3.3-70b-versatile)   <- times out at 50s
50.5s [agent] synthesis: groq_2 (llama-3.3-70b-versatile)   <- succeeds in 3s
53.5s Answer ready.
```

With 5 Groq keys and `timeout=50 s` per key for synthesis, worst-case is 250 s
before falling to Gemini. In practice one key always responds within 5–10 s.

### The Three Agent Tools

| Tool | When called | What it does |
|---|---|---|
| `search_letter_index` | Always — Step 1 | SQL query on local DB. Filters by `violation_types` (category), company name, year range. Returns ≤20 pre-classified letter dicts. Hardcapped at 20 regardless of model request. |
| `synthesize_brief` | Always — Step 2 (auto-chained) | Sends all letters + user question to Groq 70b or Gemini. Returns structured markdown brief. |
| `fetch_letter_text` | Optional — if model asks for deep analysis | Cache-first: returns `letters.full_text` from SQLite if available, else fetches live from FDA.gov. Only used when the agent needs specific sentences from a letter. |

`classify_violation` exists in `tools.py` but is **not exposed to the agent** — it is only used by the offline batch classifier (`ai/classifier.py`). All letters are already classified in the DB.

### LLM Provider Strategy

No single API key required. Three providers in priority order:

```
1. Groq (cloud, primary)
   Agent routing: llama-3.1-8b-instant
     - Typically responds in 0.3–3 s
     - timeout=90 s, max_retries=0 per key
     - 5 keys configured (GROQ_API_KEY_1 through _5), tried in order
   Synthesis:     llama-3.3-70b-versatile
     - Reads 20 letters and writes the brief
     - timeout=50 s per key

2. Ollama (local, fallback when all Groq keys fail)
   Agent routing: llama3.2  (configured via OLLAMA_AGENT_MODEL)
     - No rate limits, no cost
     - Slower on first call — Ollama may need to swap the loaded model
     - timeout=180 s (OLLAMA_TIMEOUT), max_retries=0
   Synthesis still uses Groq/Gemini (Ollama is only used for routing)

3. Gemini (cloud, last resort)
   Agent routing + synthesis: gemini-2.0-flash
     - Falls back here if both Groq and Ollama fail
     - 2 keys configured (GEMINI_API_KEY_1 through _2)
     - Retry on 503/UNAVAILABLE; raises immediately on 429/RESOURCE_EXHAUSTED
```

### Auto-chain: why no second LLM call

Earlier versions of the agent had a second LLM call after search — the model would read the search results and then decide to call `synthesize_brief`. This added 1–85 s of latency for a decision that is always the same.

The system prompt says:

> *STANDARD WORKFLOW: Step 1: call search_letter_index. Step 2: call synthesize_brief. Done. Two tool calls total.*

Since Step 2 is deterministic, `agent.py` short-circuits it: as soon as `search_letter_index` returns a non-empty list, `synthesize_brief` is called automatically. The second routing call never happens.

Benchmark (same query, same machine):

| Version | Time |
|---|---|
| Old: Ollama-first, no auto-chain, no limit cap | 250 s |
| Old: Groq-first, no auto-chain, limit=20 | 123 s |
| Current: Groq-first, auto-chain, limit=20 | **5 s** |

### Query Logging

Every query is appended to `logs/agent_YYYY-MM-DD.jsonl`:

```json
{
  "ts": "2026-06-10T14:52:28",
  "query": "What CGMP violations appeared in 2024?",
  "provider": "groq_1",
  "tool_calls": [
    {"name": "search_letter_index", "inputs": "category=CGMP, year_start=2024", "ok": true},
    {"name": "synthesize_brief",    "inputs": "user_query=What CGMP...",        "ok": true}
  ],
  "duration_s": 5.1,
  "error": null
}
```

---

## Database Schema

Single SQLite file at `data/warning_letters.db`.

```
letters
+-- id                  INTEGER PK
+-- posted_date         TEXT (MM/DD/YYYY)
+-- letter_issue_date   TEXT
+-- company_name        TEXT
+-- letter_url          TEXT UNIQUE  <- FDA.gov path
+-- issuing_office      TEXT
+-- subject             TEXT
+-- response_letter_url TEXT
+-- closeout_letter_url TEXT
+-- full_text           TEXT         <- populated by letter_fetcher
+-- fetched_at          TEXT

classifications
+-- id                  INTEGER PK
+-- letter_id           INTEGER -> letters.id
+-- violation_types     TEXT (JSON array -- canonical values only)
+-- gcp_sections        TEXT (JSON array)
+-- severity_signal     TEXT  high|medium|low
+-- therapeutic_area    TEXT
+-- company_type        TEXT
+-- key_findings        TEXT (JSON array)
+-- enforcement_context TEXT
+-- raw_json            TEXT  (full LLM response + provider label)
+-- classified_at       TEXT

scrape_runs
+-- (audit log of each index scrape)
```

---

## UI

Single-file HTML frontend (`ui/FDAWarningAITool.html`) served by a lightweight Flask backend (`server.py`). Uses SSE (Server-Sent Events) to stream agent tool steps live to the browser.

**Run:**
```
python server.py
```
Open http://localhost:8000

Three views:

**Query Agent**
- Natural language question input
- Live step-by-step progress as the agent calls tools (streamed via SSE)
- Structured markdown inspection readiness brief
- Source letter cards with company, date, severity, and violation tags

**Trends**
- 4 stat cards (total letters, classified, year range, offices)
- 6 interactive Chart.js charts — click any element to cross-filter all others:
  - Letters by Year, Top Violation Types, Issuing Office (treemap)
  - Severity Distribution (donut), Most Cited CFR/ICH Sections, Violation Trends Over Time (stacked area)
- Active filters shown as chips; "Clear all filters" button

**Letter Explorer**
- Filter by year, issuing office, violation type, severity, company type, therapeutic area, keyword
- Paginated table (20 per page, configurable) of all 3,513+ letters
- Sortable columns: date, company, office, severity
- Each row links to the original FDA.gov letter
- Expandable classification detail panel

---

## Notification Watcher

The `notify/` module watches the database for new letters matching keywords you care about and sends an email digest.

**Configure** `notify_config.yaml`:
```yaml
email:
  smtp_host: smtp.gmail.com
  smtp_port: 587
  to_addrs:
    - you@example.com

watches:
  products:   [Keytruda, Ozempic, ...]
  molecules:  [pembrolizumab, semaglutide, ...]
  companies:  [Pfizer, Novo Nordisk, ...]
```

**Run:**
```
python -m notify.watcher
```

Set `NOTIFY_EMAIL_USER` and `NOTIFY_EMAIL_PASSWORD` (Gmail App Password) in `.env`.

---

## Project Structure

```
fdawarn/
|
+-- server.py                  <- Flask entry point -> http://localhost:8000
+-- refresh.py                 <- Incremental data refresh (scrape + fetch + classify)
|
+-- scraper/
|   +-- fda_scraper.py         <- Stage 1: pull index from FDA.gov DataTables API
|   +-- letter_fetcher.py      <- Stage 2: fetch + cache full letter text
|
+-- storage/
|   +-- schema.sql             <- SQLite table definitions
|   +-- db.py                  <- Connection helper, upsert, search, stats
|
+-- ai/
|   +-- agent.py               <- Tool-calling agent loop (Groq -> Ollama -> Gemini)
|   +-- tools.py               <- Tool implementations (search, fetch, synthesize)
|   +-- classifier.py          <- Batch classifier + in-place violation normalizer
|   +-- prompts.py             <- System prompts and user templates
|   +-- agent_logger.py        <- Daily JSONL query log writer
|
+-- ui/
|   +-- FDAWarningAITool.html  <- Single-page HTML UI (Chart.js, served by server.py)
|
+-- notify/
|   +-- watcher.py             <- Keyword watcher -- scans DB, triggers email digest
|   +-- emailer.py             <- SMTP email sender
|
+-- data/
|   +-- warning_letters.db     <- SQLite database (gitignored)
|
+-- logs/
|   +-- agent_YYYY-MM-DD.jsonl <- Daily query logs (gitignored)
|
+-- context/                   <- Original design brief and UI mockup (reference only)
+-- notify_config.yaml         <- Notification keyword watches + email config
+-- .env                       <- API keys (gitignored)
+-- .env.example               <- Template -- copy to .env and fill in
+-- requirements.txt
```

---

## Setup and Running

### 1. Install dependencies

```
pip install -r requirements.txt
```

### 2. Configure API keys

Copy `.env.example` to `.env` and fill in your keys.

**Minimum required** — at least one Groq key to use the Query Agent:
```
GROQ_API_KEY_1=gsk_...    # free at console.groq.com
```

**Recommended** — 3–5 Groq keys so the agent rotates when one key hits its rate limit:
```
GROQ_API_KEY_1=gsk_...
GROQ_API_KEY_2=gsk_...
GROQ_API_KEY_3=gsk_...
GROQ_API_KEY_4=gsk_...
GROQ_API_KEY_5=gsk_...
GEMINI_API_KEY_1=AI...    # free at aistudio.google.com — last-resort fallback
GEMINI_API_KEY_2=AI...
```

**Optional** — Ollama for local inference (no internet needed after models are pulled):
```
OLLAMA_BASE_URL=http://localhost:11434/v1
OLLAMA_MODEL=gemma4          # used for offline classification
OLLAMA_AGENT_MODEL=llama3.2  # used for agent routing when all Groq keys fail
OLLAMA_TIMEOUT=180           # seconds — allows for model swap overhead on first call
```

### 3. Run the data pipeline (one time, then refresh as needed)

**First-time setup** — scrapes all 3,500+ letters, fetches their full text, and classifies them. Takes several hours (mostly the text-fetch step):

```
python -m scraper.fda_scraper        # ~1 min — fetches full index
python -m scraper.letter_fetcher     # ~3 hrs — downloads all letter text
python -m ai.classifier              # ~2 hrs — classifies with AI
```

**Incremental refresh** — run ad-hoc whenever you want to pick up new letters. Each stage skips work already done:

```
python refresh.py
```

This chains all three stages automatically:
- Stage 1: fetches FDA.gov index but **stops paginating** as soon as it hits a page of already-known letters (typically 1–2 AJAX requests instead of 7+)
- Stage 2: fetches full text only for letters with `full_text IS NULL`
- Stage 3: classifies only letters with no classification or incomplete fields

If FDA publishes 5 new letters since your last run, `refresh.py` typically completes in under 2 minutes (network time for the new letters + one classify call per new letter).

Options:
```
python refresh.py --scrape-only    # just check for new letters in the index
python refresh.py --no-classify    # scrape + fetch text, skip AI classify step
python refresh.py --dry-run        # print what each stage would do without running
```

You can also run each stage individually with its full options:
```
python -m scraper.fda_scraper --incremental   # incremental index scrape only
python -m scraper.letter_fetcher --year 2025  # fetch text for 2025 letters only
python -m ai.classifier --year 2025           # classify 2025 letters only
```

### 4. Launch the app

```
python server.py
```
Open http://localhost:8000

---

## Key Design Decisions

**Why SQLite?** Zero-setup, single-file database. The full dataset (3,500 letters, all full text) fits comfortably. WAL mode allows concurrent reads during background fetching.

**Why scrape the AJAX endpoint instead of HTML?** The FDA.gov warning letter page uses Drupal DataTables which loads data via a JSON AJAX call. Hitting that endpoint directly returns all 3,500+ records in clean JSON batches, far more reliable than parsing rendered HTML.

**Why no LangChain or similar framework?** The agent loop is simple enough (2 tools in the standard workflow) that a custom loop is cleaner, more debuggable, and has no external dependencies beyond the LLM SDKs.

**Why Groq first, not Ollama?** Groq 8b-instant responds in 0.3–3 s on normal days. Local Ollama takes 60–180 s for the first routing call because it swaps the resident model (gemma4, 9.6 GB) to llama3.2 before inference. On a dedicated GPU with the right model loaded, Ollama would be faster — but in practice Groq wins for routing. Ollama is kept as a fallback for when all Groq keys are exhausted.

**Why auto-chain instead of letting the model call synthesize_brief?** Benchmark showed that letting the 8b model decide to call synthesize_brief added one full LLM round-trip (1–85 s depending on Groq load). Since the workflow is fixed — search always precedes synthesis — the code short-circuits that decision. See the benchmark table in the AI Agent section.

**Why cache everything in SQLite?** FDA.gov has robots.txt restrictions and rate limits. Caching full text means the agent never needs to refetch a letter it has already read. Classifications are also cached — repeated queries about the same letter cost nothing.

**Why a keyword normalizer instead of re-classifying?** When violation categories change or LLM output drifts, `--normalize` repairs all 3,500+ rows in seconds using a local keyword map, without spending any API tokens.

---

## Data Source

All data is sourced from FDA.gov public records:
- Warning letter index: https://www.fda.gov/inspections-compliance-enforcement-and-criminal-investigations/compliance-actions-and-activities/warning-letters
- Individual letter pages: linked from the index

No proprietary data. No login required. All content is US government public domain.
