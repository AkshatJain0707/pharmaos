# PharmaOS Intelligence Platform

> **Production-grade, multi-domain pharmaceutical competitive intelligence pipeline — powered by n8n, LLMs, and PostgreSQL.**

[![Pipeline Version](https://img.shields.io/badge/Pipeline-v5.0.0-brightgreen)]()
[![Built with n8n](https://img.shields.io/badge/Built%20with-n8n-EA4B71)](https://n8n.io)
[![PostgreSQL](https://img.shields.io/badge/Database-PostgreSQL-336791)](https://www.postgresql.org)
[![OpenRouter](https://img.shields.io/badge/LLM-OpenRouter-black)](https://openrouter.ai)
[![Docker](https://img.shields.io/badge/Deploy-Docker-2496ED)](https://www.docker.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

---

## What is PharmaOS?

PharmaOS is a self-hostable pharmaceutical intelligence platform that ingests raw signals from clinical trial registries, regulatory filings, academic publications, and competitor pipelines — then transforms them into structured, AI-narrated daily intelligence digests delivered to executives, medical affairs, and domain teams.

It is built entirely on **n8n** (no-code/low-code workflow orchestration), **PostgreSQL**, and **OpenRouter LLMs** — with zero vendor lock-in and full auditability on every run.

---

## Live Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        PharmaOS Pipeline v5.0.0                         │
│                                                                         │
│   WF1 — Ingestion          WF2 — Scoring           WF3 — Intelligence   │
│   ──────────────           ─────────────           ───────────────────  │
│   ClinicalTrials.gov  ──▶  Impact scoring    ──▶   Domain briefings     │
│   PubMed / bioRxiv         Route tier assign        LLM summaries       │
│   Competitor sites         Compound detection       Action items        │
│   Regulatory feeds         Competitor flags         Watch lists         │
│   Manual webhook           Confidence scores        CSV export          │
│                            Deduplication                │               │
│                                                         ▼               │
│                          WF4 — Daily Digest & Notification Router       │
│                          ──────────────────────────────────────────     │
│                                                                         │
│          ┌──────────────┬──────────────┬────────────┬──────────────┐    │
│          │ HTML Email   │ Slack Exec   │ Slack Ops  │  PagerDuty   │    │
│          │ (C-Suite +   │ Block Kit    │ Telemetry  │  v2 Events   │    │
│          │  Med Affairs)│ Digest       │ Dashboard  │  (Critical)  │    │
│          └──────────────┴──────────────┴────────────┴──────────────┘    │
│                                     │                                   │
│                          pharmaos_audit_log (PostgreSQL)                │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Key Features

### 🔬 Signal Intelligence Engine
- Multi-source ingestion across ClinicalTrials.gov, PubMed, bioRxiv, regulatory feeds, and competitor sites
- Automated impact scoring with 4-tier route classification (`LOW → MED → HIGH → CRITICAL`)
- Compound event detection — correlates signals across domains to surface emergent risks
- Competitor flag detection, urgency classification, and confidence scoring
- Signal deduplication by URL fingerprint and semantic hash
- Full signal age and velocity tracking

### 🧠 AI Narrative Engine (3-Model Architecture)
- **GPT-4.1 (flagship)** — C-suite executive summary, top threat, top opportunity, cross-domain insights, week-ahead outlook
- **GPT-4.1-mini** — 4 parallel domain briefs (CI, Safety, Regulatory, BD), each audience-calibrated
- Structured JSON output with strict schema enforcement and full fallback handling on parse errors
- Safety signals always take priority — CMO-specific safety alert field
- Token usage tracked per run and written to `pharmaos_system_health`

### 📡 Multi-Channel Dispatch (5 Channels)
| Channel | Audience | Content |
|---|---|---|
| `#pharmaos-executive` | C-Suite | Full Block Kit digest with risk score bar, threat/opportunity cards, compound event alerts |
| `#pharmaos-ci/safety/regulatory/bd` | Domain teams | Domain-specific briefings with action items, signal cards, watch lists |
| `#pharmaos-ops` | Engineering | Run telemetry, token usage, velocity breakdown, blind spots, SLA status |
| HTML Email | VP Med Affairs + C-Suite | Branded HTML with metrics dashboard, domain tables, top signals, CSV attachment |
| PagerDuty | On-call | v2 Events API with dedup key, severity, custom details — fires only on EXECUTIVE escalations |

### 📊 14-Factor Analytics Engine
Every run computes: global metrics, per-domain breakdown, day-over-day trends, compound event analysis, indication heatmap, geographic signal density, velocity buckets, asset coverage gap detection, composite risk score (0–100), strategy memory overlay, pipeline health metrics, signal type distribution, source distribution, and top drugs by activity.

### 🔒 Production-Grade Reliability
- Every dispatch node: `retryOnFail: true`, 2 retries, 5s backoff
- SLA health check after every run (target: < 90 seconds end-to-end)
- AI parse error fallback — workflow never silently fails
- Full audit trail on every run written to `pharmaos_audit_log`
- PagerDuty dedup key prevents duplicate incidents on retriggers
- Materialized view refresh after every digest

---

## Tech Stack

| Layer | Technology |
|---|---|
| Workflow Orchestration | n8n (self-hosted) |
| Database | PostgreSQL 14+ |
| LLM APIs | OpenRouter (GPT-4.1, GPT-4.1-mini) |
| Signal Ingestion | Scrapy / FastAPI |
| Email Dispatch | Gmail / SMTP |
| Messaging | Slack Block Kit (`chat.postMessage`) |
| Alerting | PagerDuty Events API v2 |
| Containerization | Docker / Docker Compose |
| Code Nodes Runtime | Node.js (n8n sandboxed V8) |

---

## Workflow Reference

### WF1 — Ingestion & Normalization
Pulls signals from external APIs and scrapers, normalizes to canonical schema, deduplicates by URL and semantic hash, and writes to `signals_mesh`.

### WF2 — Scoring, Routing & Enrichment
Computes `impact_level`, `priority_score`, `route_tier`, `confidence`, and `is_urgent` for every signal. Detects compound events across correlated signals. Flags competitor signals. Maintains full override audit trail.

### WF3 — Domain Intelligence Builders
Groups signals by domain, calls LLM per domain for structured briefings, produces domain analytics, metrics tables, and CSV exports. Stores results in `pharmaos_domain_briefings`.

### WF4 — Daily Digest & Notification Router `v5.0.0`
**43 nodes, 7 phases, 3 AI models, 5 output channels.**

| Phase | Nodes | Function |
|---|---|---|
| 1 | 1–12 | Trigger, context build, 8 parallel SQL queries, merge |
| 2 | 13 | 14-factor analytics computation |
| 3 | 14–22 | AI prompt construction + 3 parallel AI calls + parsing |
| 4 | 23–27 | Output assembly (Slack blocks, HTML email, CSV, PagerDuty) |
| 5 | 28–35 | Multi-channel dispatch |
| 6 | 36–38 | Audit log, health metrics, materialized view refresh |
| 7 | 39–43 | SLA health check, ops alert, terminal |

**Adaptive Scheduling — 3 trigger modes:**
| Trigger | Condition | Window | Extras |
|---|---|---|---|
| Daily | CRON 8:00 AM | 24h primary, 7d trend | Slack + Email |
| Adaptive | WF2 calls WF4 on EXECUTIVE escalation | 6h primary, 24h trend | + PagerDuty |
| Weekly | Monday auto-detect | 7d primary, 30d trend | + PDF + Drive |

---

## Database Schema (Core Tables)

```sql
-- Primary signal store
CREATE TABLE signals_mesh (
  id                     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  source_id              VARCHAR(255),
  signal_domain          VARCHAR(80),
  signal_type            VARCHAR(80),
  source                 VARCHAR(120),
  title                  TEXT,
  summary                TEXT,
  drug_name              VARCHAR(200),
  entity_name            VARCHAR(200),
  indication             VARCHAR(200),
  sponsor                VARCHAR(200),
  phase                  VARCHAR(50),
  impact_level           VARCHAR(20),
  impact_score           NUMERIC(5,2),
  priority_score         NUMERIC(5,2),
  route_tier             VARCHAR(40),
  competitor_flag        BOOLEAN DEFAULT FALSE,
  is_urgent              BOOLEAN DEFAULT FALSE,
  confidence             VARCHAR(20),
  ai_summary             TEXT,
  recommended_action     TEXT,
  override_status        VARCHAR(40),
  override_by            VARCHAR(100),
  override_reason        TEXT,
  asset_id               VARCHAR(100),
  version                INTEGER DEFAULT 1,
  quality_score          NUMERIC(5,2),
  safety_severity        VARCHAR(40),
  patient_impact_estimate TEXT,
  fingerprint            VARCHAR(64),
  dedup_hash             VARCHAR(64),
  pipeline_version       VARCHAR(20),
  url                    TEXT,
  nct_id                 VARCHAR(20),
  geography              VARCHAR(120),
  first_seen_at          TIMESTAMPTZ DEFAULT NOW(),
  last_seen_at           TIMESTAMPTZ DEFAULT NOW(),
  ai_processed           BOOLEAN DEFAULT FALSE
);

-- Compound event store
CREATE TABLE compound_events (
  id                   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  title                TEXT,
  domains              TEXT[],
  signal_ids           BIGINT[],
  compound_risk_score  NUMERIC(5,2),
  trigger_pattern      VARCHAR(80),
  pattern_metadata     JSONB DEFAULT '{}',
  ai_narrative         TEXT,
  recommended_actions  TEXT,
  escalation_level     VARCHAR(40),
  resolved             BOOLEAN DEFAULT FALSE,
  false_positive       BOOLEAN DEFAULT FALSE,
  detected_at          TIMESTAMPTZ DEFAULT NOW()
);

-- Full audit trail
CREATE TABLE pharmaos_audit_log (
  id               BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  event_type       VARCHAR(80)  NOT NULL,
  domain           VARCHAR(50)  DEFAULT '',
  actor            VARCHAR(100) DEFAULT 'system',
  action           TEXT         DEFAULT '',
  new_state        JSONB        DEFAULT '{}',
  metadata         JSONB        DEFAULT '{}',
  pipeline_version INTEGER      DEFAULT 5,
  created_at       TIMESTAMPTZ  DEFAULT NOW()
);

-- Pipeline health & KPI metrics
CREATE TABLE pharmaos_system_health (
  id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  metric_name   VARCHAR(120) NOT NULL,
  metric_value  NUMERIC,
  unit          VARCHAR(40),
  tags          JSONB        DEFAULT '{}',
  recorded_at   TIMESTAMPTZ  DEFAULT NOW()
);
```

---

## Quick Start

### Prerequisites
- Docker & Docker Compose
- n8n self-hosted (v1.x+)
- PostgreSQL 14+
- Slack App with `chat:write` scope
- PagerDuty Events v2 integration key
- OpenRouter API key

### 1. Clone the repository

```bash
git clone https://github.com/yourusername/pharmaos.git
cd pharmaos
```

### 2. Configure environment

```bash
cp .env.example .env
```

Edit `.env`:

```env
# ── Database ──────────────────────────────────────
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=pharmaosdb
POSTGRES_USER=pharmaos
POSTGRES_PASSWORD=your_secure_password

# ── LLM ───────────────────────────────────────────
OPENROUTER_API_KEY=sk-or-your-key-here

# ── Slack ─────────────────────────────────────────
SLACK_BOT_TOKEN=xoxb-your-bot-token

# ── PagerDuty ─────────────────────────────────────
PAGERDUTY_ROUTING_KEY=your-pd-v2-integration-key

# ── Email ─────────────────────────────────────────
SMTP_HOST=smtp.yourprovider.com
SMTP_PORT=587
SMTP_USER=no-reply@pharmaos.internal
SMTP_PASS=your_smtp_password

# ── Recipients (comma-separated for multiple) ─────
PHARMAOS_EMAIL_CSUITE=ceo@pharma.com,cmo@pharma.com
PHARMAOS_EMAIL_MED_AFFAIRS=vp-ma@pharma.com
PHARMAOS_EMAIL_SAFETY=pv-director@pharma.com
PHARMAOS_EMAIL_REGULATORY=reg-vp@pharma.com
PHARMAOS_EMAIL_BD=bd-head@pharma.com
PHARMAOS_EMAIL_OPS=ops@pharma.com

# ── Slack Channels ────────────────────────────────
PHARMAOS_SLACK_EXECUTIVE=#pharmaos-executive
PHARMAOS_SLACK_CI=#pharmaos-ci-team
PHARMAOS_SLACK_SAFETY=#pharmaos-safety
PHARMAOS_SLACK_REGULATORY=#pharmaos-medical-affairs
PHARMAOS_SLACK_BD=#pharmaos-bd-team
PHARMAOS_SLACK_OPS=#pharmaos-ops
```

### 3. Start services

```bash
docker compose up -d
```

### 4. Initialize the database

```bash
psql -U pharmaos -d pharmaosdb -f sql/PharmaOS_Schema.sql
```

### 5. Import n8n workflows

In your n8n instance go to **Workflows → Import from file** and import in order:

```
workflows/
  WF1_Ingestion_Normalization.json
  WF2_Scoring_Routing_Enrichment.json
  WF3_Domain_Intelligence_Builders.json
  WF4_Daily_Digest_Dispatch_v500.json
```

### 6. Set n8n Variables

In n8n go to **Settings → Variables** and add all keys from your `.env` file. The workflow reads recipients, channels, and AI model config from `$vars` at runtime.

### 7. Activate WF4

Set WF4 to **Active** — it will fire automatically at 8:00 AM daily and auto-detect weekly mode every Monday.

---

## n8n Variables Reference

Set these in **Settings → Variables** inside your n8n instance:

| Variable | Required | Description |
|---|---|---|
| `PHARMAOS_EMAIL_CSUITE` | ✅ | CEO/CMO email(s), comma-separated |
| `PHARMAOS_EMAIL_MED_AFFAIRS` | ✅ | VP Medical Affairs email(s) |
| `PHARMAOS_EMAIL_SAFETY` | ✅ | PV/Safety team email(s) |
| `PHARMAOS_EMAIL_REGULATORY` | ✅ | Regulatory Affairs email(s) |
| `PHARMAOS_EMAIL_BD` | ✅ | BD team email(s) |
| `PHARMAOS_EMAIL_OPS` | ✅ | Ops/Engineering email(s) |
| `PHARMAOS_SLACK_EXECUTIVE` | ✅ | e.g. `#pharmaos-executive` |
| `PHARMAOS_SLACK_CI` | ✅ | e.g. `#pharmaos-ci-team` |
| `PHARMAOS_SLACK_SAFETY` | ✅ | e.g. `#pharmaos-safety` |
| `PHARMAOS_SLACK_REGULATORY` | ✅ | e.g. `#pharmaos-medical-affairs` |
| `PHARMAOS_SLACK_BD` | ✅ | e.g. `#pharmaos-bd-team` |
| `PHARMAOS_SLACK_OPS` | ✅ | e.g. `#pharmaos-ops` |
| `PHARMAOS_AI_EXEC_MODEL` | ⬜ | Default: `openai/gpt-4.1` |
| `PHARMAOS_AI_DOMAIN_MODEL` | ⬜ | Default: `openai/gpt-4.1-mini` |
| `PAGERDUTY_ROUTING_KEY` | ✅ | PagerDuty v2 Events integration key |

---

## KPI Targets

| KPI | Target |
|---|---|
| Digest generation success rate | > 99% |
| End-to-end dispatch latency | < 90 seconds |
| HIGH signal coverage ratio | > 95% |
| AI narrative confidence (high%) | > 60% |
| Override compliance rate | < 5% violations |
| Compound event → executive alert | < 15 minutes |
| Channel delivery rate | 100% (with retry) |

---

## Project Structure

```
pharmaos/
├── workflows/                  # n8n workflow JSON exports
│   ├── WF1_Ingestion_Normalization.json
│   ├── WF2_Scoring_Routing_Enrichment.json
│   ├── WF3_Domain_Intelligence_Builders.json
│   └── WF4_Daily_Digest_Dispatch_v500.json
├── sql/
│   └── PharmaOS_Schema.sql     # Full PostgreSQL schema
├── scraper/                    # Scrapy-based pharma signal scraper
├── api/                        # FastAPI backend services
├── docker-compose.yml
├── .env.example
└── README.md
```

---

## Roadmap

- [ ] WF5 — Human-in-the-loop signal override interface
- [ ] Real-time signal streaming via webhook triggers
- [ ] DSMB briefing agent integration
- [ ] Regulatory filing tracker (FDA ANDA/NDA, EMA, CDSCO)
- [ ] Multi-tenant audience routing
- [ ] ML confidence scoring model (replace rule-based)
- [ ] Signal clustering by therapeutic mechanism (MoA grouping)
- [ ] Dashboard UI (Next.js)
- [ ] Weekly PDF briefing with Drive upload

---

## Contributing

Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change. All code nodes must be tested against the n8n sandboxed V8 environment — `process`, `require`, and `fs` are not available.

---

## License

[MIT](LICENSE)

## Author

Built by **Akshat Sunil Jain**

> *PharmaOS gives pharmaceutical intelligence teams the same signal infrastructure that top-tier CI operations use internally — as a fully open, self-hostable platform.*
