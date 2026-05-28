# Competitor Intelligence Platform — Project Plan

## What This File Is

This is the global context file for AI assistants (GitHub Copilot, Claude, Cursor).
It contains system architecture, coding rules, and the full product roadmap.
**We are currently building V1 only.** Do not suggest V2/V3 features unless explicitly asked.

---

## Product Overview

An AI-powered competitor intelligence platform that monitors competitor activity (starting with LinkedIn), extracts strategic signals, builds long-term memory, and generates actionable reports.

This is NOT a simple LLM summarizer. It is:
- Event-driven (async workers, never block the request lifecycle)
- Memory-centric (short, mid, long-term memory per competitor)
- AI-first (structured JSON extraction, embeddings, semantic search)
- Modular and production-grade from day one

**Primary goal:** Convert raw competitor activity into structured strategic intelligence.

---

## Roadmap Overview (Build V1 First)

### V1 — "It Actually Works" ← WE ARE HERE
Core ingestion → enrichment → dashboard loop.
Prove the intelligence quality before adding complexity.

### V2 — "It Has Memory" (future)
Short-term (14d) and mid-term (90d) memory per competitor.
Novelty scoring, topic clustering, weekly intelligence reports.
Do NOT build any of this in V1.

### V3 — "It Thinks Strategically" (future)
Long-term memory, semantic drift detection, strategic scoring system,
cross-competitor market intelligence, anomaly alerts.
Do NOT build any of this in V1.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js, Tailwind CSS, React Query |
| Backend | FastAPI, Python 3.12+ |
| Database | PostgreSQL + pgvector |
| Queue | Redis + Celery |
| AI | OpenAI (gpt-4o-mini for enrichment, text-embedding-3-small for embeddings) |
| Scraping | Proxycurl or Apify (behind abstract adapter), RSS/Google News fallback |
| Deployment | Docker + Docker Compose (single VPS) |
| Migrations | Alembic |
| Logging | Structured JSON logs (not print statements) |

---

## Folder Structure

```
backend/
├── app/
│   ├── core/
│   │   ├── config.py          # Pydantic settings, env vars
│   │   ├── database.py        # Async SQLAlchemy engine + session
│   │   ├── logging.py         # Structured JSON logger setup
│   │   └── constants.py       # Enums, shared constants
│   │
│   ├── modules/
│   │   ├── competitors/       # CRUD, schema, service, router
│   │   ├── activities/        # Raw activity ingestion
│   │   ├── enrichment/        # Enrichment results storage
│   │   └── reports/           # Daily digest
│   │
│   ├── ai/
│   │   ├── prompts/           # One file per prompt, versioned
│   │   ├── providers/         # LLM client wrappers
│   │   ├── embeddings/        # Embedding generation
│   │   └── classifiers/       # Signal classification logic
│   │
│   ├── workers/
│   │   ├── celery_app.py      # Celery instance
│   │   ├── scheduler.py       # Celery beat schedule
│   │   └── tasks/             # One file per task domain
│   │
│   └── infrastructure/
│       ├── ingestion/         # Scraper adapters (abstract interface)
│       ├── vectorstore/       # pgvector helpers
│       └── llm/               # Raw HTTP/SDK LLM client
│
├── alembic/                   # DB migrations
├── tests/
├── scripts/
│   └── eval_prompt.py         # Prompt evaluation runner
├── docker/
└── pyproject.toml

frontend/
├── app/                       # Next.js app router
├── components/
├── lib/
│   └── api.ts                 # API client (React Query hooks)
└── types/
```

---

## V1 Database Schema (Only These Tables)

```sql
-- Core competitor record
competitors (
  id            UUID PRIMARY KEY,
  name          TEXT NOT NULL,
  linkedin_url  TEXT UNIQUE NOT NULL,
  domain        TEXT,
  industry      TEXT,
  priority      TEXT DEFAULT 'medium',  -- low | medium | high
  status        TEXT DEFAULT 'active',  -- active | paused | deleted
  created_at    TIMESTAMPTZ DEFAULT now(),
  updated_at    TIMESTAMPTZ DEFAULT now()
)

-- Raw scraped content — never overwrite, never delete
activities (
  id            UUID PRIMARY KEY,
  competitor_id UUID REFERENCES competitors(id),
  raw_content   TEXT NOT NULL,
  content_hash  TEXT UNIQUE NOT NULL,   -- SHA256 for dedup
  source        TEXT NOT NULL,          -- linkedin | rss | manual
  post_date     TIMESTAMPTZ,
  scraped_at    TIMESTAMPTZ DEFAULT now()
)

-- AI enrichment output — always additive, never overwrite raw
enriched_activities (
  id                    UUID PRIMARY KEY,
  activity_id           UUID REFERENCES activities(id) UNIQUE,
  signal_type           TEXT,           -- product_launch | hiring | partnership | thought_leadership | other
  topics                TEXT[],
  technologies          TEXT[],
  importance            INTEGER,        -- 1-5
  confidence            FLOAT,          -- 0.0-1.0
  summary               TEXT,
  strategic_implication TEXT,
  prompt_version        TEXT NOT NULL,
  created_at            TIMESTAMPTZ DEFAULT now()
)

-- Embeddings in separate table for clean indexing
activity_embeddings (
  id          UUID PRIMARY KEY,
  activity_id UUID REFERENCES activities(id) UNIQUE,
  embedding   vector(1536) NOT NULL,    -- text-embedding-3-small
  created_at  TIMESTAMPTZ DEFAULT now()
)

-- Every LLM call logged — never skip this
llm_logs (
  id             UUID PRIMARY KEY,
  prompt_version TEXT NOT NULL,
  input_hash     TEXT,
  raw_output     TEXT,
  parsed_ok      BOOLEAN,
  latency_ms     INTEGER,
  tokens_used    INTEGER,
  cost_usd       FLOAT,
  created_at     TIMESTAMPTZ DEFAULT now()
)

-- Daily digest reports
reports (
  id           UUID PRIMARY KEY,
  report_type  TEXT DEFAULT 'daily',
  report_json  JSONB NOT NULL,
  generated_at TIMESTAMPTZ DEFAULT now()
)
```

---

## V1 API Endpoints

```
POST   /competitors                    # Add competitor by LinkedIn URL → enqueue scrape job
GET    /competitors                    # List with last_activity_at, signal_count
GET    /competitors/{id}               # Full profile + recent enriched activities
PATCH  /competitors/{id}               # Update priority, tags
DELETE /competitors/{id}               # Soft delete

POST   /activities/manual              # Manually paste content for enrichment (for testing)
GET    /activities?competitor_id=&signal_type=   # Filtered activity feed

GET    /reports/daily                  # Latest daily digest
GET    /reports/daily?date=YYYY-MM-DD  # Specific date digest
```

---

## Data Pipeline (V1)

```
User adds competitor URL
        ↓
POST /competitors → validate → save → enqueue scrape job → return 202
        ↓
[Celery Worker] scrape_competitor_task
  → fetch LinkedIn page via Proxycurl/Apify
  → fallback to RSS/Google News if scraper fails
  → dedup on content_hash
  → save raw to activities table
  → enqueue enrichment job
        ↓
[Celery Worker] enrich_activity_task
  → lightweight pre-filter (length check, language, keyword heuristic)
  → skip noise without LLM call
  → generate embedding → save to activity_embeddings
  → call LLM with signal extraction prompt
  → validate JSON output with Pydantic
  → retry once on parse failure
  → save to enriched_activities
  → log to llm_logs
        ↓
[Celery Beat — 07:00 daily] generate_daily_digest_task
  → pull enriched activities from last 24h
  → rank by importance score
  → call LLM to write digest JSON
  → save to reports table
```

---

## AI Layer Rules

### Prompt Files
Every prompt lives in `/app/ai/prompts/` as a Python file.
Each file must export exactly:

```python
VERSION = "signal_extraction_v1"

PROMPT_TEMPLATE = """..."""

class OutputSchema(BaseModel):
    signal_type: Literal["product_launch", "hiring", "partnership", "thought_leadership", "other"]
    topics: list[str]
    technologies: list[str]
    importance: int          # 1-5
    confidence: float        # 0.0-1.0
    summary: str
    strategic_implication: str
```

### LLM Output Rules
- LLMs must return structured JSON only — no prose, no preamble
- Always validate output with Pydantic schema
- On validation failure: retry once, then log failure and skip
- Log every LLM call to `llm_logs` — no exceptions
- Store `prompt_version` on every `enriched_activities` row

### Embedding Rules
- Use `text-embedding-3-small` (1536 dims, cheap, fast)
- Store in `activity_embeddings`, not on the activities row
- Generate embedding before LLM enrichment (cheaper to fail early)

---

## Coding Standards

### General
- Python 3.12+, type hints everywhere, no untyped functions
- Use `async`/`await` throughout — no blocking I/O in async context
- Pydantic v2 for all data validation and settings
- Keep functions small and single-purpose
- No business logic in route handlers — routes validate, enqueue, return

### FastAPI Routes Must
- Validate input (Pydantic schemas)
- Enqueue work via Celery
- Return quickly (202 Accepted for async jobs)
- Never run AI analysis or long operations inline

### Celery Workers Must
- Be idempotent — safe to run the same task twice
- Log every step with structured JSON
- Support reprocessing any activity from scratch
- Handle failures gracefully — catch, log, don't crash the worker
- Use exponential backoff for retries (max 3 attempts)

### Database
- Use async SQLAlchemy with asyncpg driver
- All timestamps in UTC (TIMESTAMPTZ)
- Never delete raw data — use soft deletes and status flags
- All enrichment is additive — new columns, new rows, never overwrites
- Index: `activities.content_hash`, `activities.competitor_id`, `activities.scraped_at`
- Index: `enriched_activities.signal_type`, `enriched_activities.importance`
- HNSW index on `activity_embeddings.embedding`

### Error Handling
- Use custom exception classes in `app/core/exceptions.py`
- Never let a worker crash silently — always log with context
- External API failures (scraper, LLM) must be caught and retried
- Validate all external content before storing (sanitize inputs)

### Logging
- Use structured JSON logger everywhere — never use `print()`
- Every log entry must include: `timestamp`, `level`, `module`, `event`
- Workers log: task_id, competitor_id, activity_id at each step
- LLM calls log: prompt_version, latency_ms, tokens, cost_usd, parsed_ok

---

## Environment Variables

```env
# Database
DATABASE_URL=postgresql+asyncpg://user:pass@localhost:5432/compintel

# Redis
REDIS_URL=redis://localhost:6379/0

# AI
OPENAI_API_KEY=sk-...
OPENAI_EMBEDDING_MODEL=text-embedding-3-small
OPENAI_ENRICHMENT_MODEL=gpt-4o-mini

# Scraping
PROXYCURL_API_KEY=...
APIFY_API_KEY=...

# App
ENVIRONMENT=development   # development | production
LOG_LEVEL=INFO
SECRET_KEY=...
```

---

## What to Build in V1 (Strict Scope)

✅ Project setup, Docker Compose, Alembic migrations
✅ Competitor CRUD API
✅ Activity ingestion (Proxycurl + RSS fallback + manual paste)
✅ Deduplication on content hash
✅ Enrichment pipeline (pre-filter → embed → LLM extract → store)
✅ Prompt versioning + eval script
✅ LLM call logging
✅ Daily digest generation (Celery beat)
✅ Dashboard: add competitor, competitor list, activity feed, critical updates, daily digest

---

## What NOT to Build in V1

❌ Memory engine (short-term, mid-term, long-term) — V2
❌ Novelty scoring — V2
❌ Topic clustering — V2
❌ Weekly reports — V2
❌ Trend detection — V2
❌ Semantic drift detection — V3
❌ Strategic scoring (AI Adoption Score etc.) — V3
❌ Cross-competitor market intelligence — V3
❌ Anomaly detection — V3
❌ Full auth system (add basic API key auth only if sharing access)

If asked to implement anything from this list, respond:
"That's a V2/V3 feature. Out of scope for V1. Do you want to add it to the backlog?"

---

## V1 Frontend Screens

### 1. Dashboard (Home)
- Critical Updates section — importance ≥ 4, last 48h, max 8 items
- Recent Activity feed — all competitors, reverse chronological, filterable by signal_type

### 2. Add Competitor
- Input: LinkedIn URL
- Auto-fetch: company name, industry, logo
- Trigger scrape + enrichment job on submit
- Show job status (queued → processing → done)

### 3. Competitors List
- Columns: name, logo, industry, priority, last activity date, signal count, status

### 4. Competitor Detail
- Header: name, logo, domain, industry, priority badge
- Activity timeline: reverse chronological, filterable by signal_type
- Signal type badge + importance score on each activity item

### 5. Daily Digest
- Latest digest by default
- Date picker for historical digests
- Structured readable sections: top signals, key competitors active today

---

## Prompt Evaluation

Before shipping any new or updated prompt:

```bash
# Run eval against ground truth dataset
python scripts/eval_prompt.py --prompt signal_extraction_v2

# Output: accuracy score, failure cases, cost estimate
# Only ship if score >= previous version
```

Ground truth dataset: `scripts/eval_data/signal_extraction_ground_truth.json`
Minimum 50 labelled examples before running evals.
Store which prompt version was used on every `enriched_activities` row.