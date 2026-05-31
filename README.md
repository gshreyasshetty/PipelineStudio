# ⚡ PipelineStudio

**End-to-end data pipeline builder — from API ingestion to Snowflake warehouse to live monitoring.**

PipelineStudio automates the full lifecycle of building Azure Function-based data pipelines. Engineers use a guided web UI to configure, generate, test, and deploy production-ready pipelines in minutes — replacing hours of manual scripting. A companion Streamlit-based Monitoring Dashboard provides real-time observability over pipeline health, data freshness, and ingestion metrics across all environments.

---

## 🧠 The Problem

Enterprise IT and InfoSec teams pull data from dozens of external APIs (Microsoft Graph, Defender, BitSight, Armis, etc.) into Snowflake for security analytics and compliance. Each new data source requires:

1. A **Timer-triggered Azure Function** (PowerShell) — auth, pagination, blob upload
2. A **Snowflake pipeline** — External Stage → Raw Table → COPY Procedure → Typed View
3. **Testing** — validate the function runs and produces correct output
4. **Monitoring** — ensure pipelines stay healthy across DEV / UAT / PRD

PipelineStudio replaces this repetitive 4-step manual process with a single guided workflow.

![PipelineStudio Overview](readme/pipeline-studio-overview.png)

---

## 🏗️ Architecture

```
┌─────────────────────────── PipelineStudio ───────────────────────────┐
│                                                                      │
│  React Frontend (Vite)          Express Backend (Node.js)            │
│  ┌──────────────────┐           ┌──────────────────────────┐        │
│  │ 6-Tab Wizard UI  │◄─REST───►│ Template Engine (HBS)     │        │
│  │ Configure → Gen  │  + WS    │ AI Generator (OpenAI/GH)  │        │
│  │ → Test → Report  │          │ Snowflake SQL Generator   │        │
│  │ → Snowflake SQL  │          │ Function Tester (func CLI)│        │
│  └──────────────────┘           └──────────┬───────────────┘        │
│                                             │ Generates              │
└─────────────────────────────────────────────┼────────────────────────┘
                                              ▼
              ┌──────────────────────────────────────────────┐
              │         Generated Artifacts                   │
              │  • run.ps1        (PowerShell function)       │
              │  • function.json  (Timer trigger binding)     │
              │  • snowflake.sql  (Full pipeline DDL)         │
              └──────────┬───────────────────┬───────────────┘
                         ▼                   ▼
              ┌──────────────────┐  ┌────────────────────┐
              │  Azure Functions │  │  Snowflake          │
              │  Timer Trigger   │  │  Stage → Raw Table  │
              │       │          │  │  → COPY Proc → View │
              │       ▼          │  │  → Scheduled Task   │
              │  Azure Blob      │  │       │             │
              │  Storage ────────┼──┼──►External Stage    │
              └──────────────────┘  └────────┬───────────┘
                                             ▼
                                  ┌──────────────────────┐
                                  │  Monitoring Dashboard │
                                  │  (Snowflake Streamlit)│
                                  │  Pipeline Health ×    │
                                  │  3 Environments       │
                                  └──────────────────────┘
```
---

## 🔄 How It Works — 6-Step Workflow

| Step | Tab | What Happens |
|------|-----|-------------|
| **1** | **Configure** | Fill a guided form — function name, API URL, auth type (OAuth / API Key / Basic / GraphQL), blob container, CRON schedule, pagination strategy |
| **2** | **Generated Code** | Backend generates `run.ps1` + `function.json` using Handlebars templates or a Copilot prompt file. Preview with syntax highlighting |
| **3** | **Live Testing** | Spawns `func host start`, triggers the function N times, streams output via WebSocket to a real-time console |
| **4** | **Report** | Automated test report — success rate, avg/min/max duration, blob write verification, error analysis, PASS/FAIL verdict |
| **5** | **Snowflake Setup** | Configure database, schema, object names. Paste sample JSON → AI-powered schema detection auto-infers column types and LATERAL FLATTEN for nested arrays |
| **6** | **Snowflake SQL** | Generates complete DDL: External Stage → Raw Table → COPY Procedure → Typed View → optional Task. Visual pipeline diagram + migration file naming |



---

## ⚙️ Code Generation Modes

PipelineStudio supports **3 generation modes** depending on preference:

| Mode | How It Works | Best For |
|------|-------------|----------|
| **Template** | Deterministic Handlebars rendering — no AI, no API key needed | Standard auth patterns (Graph OAuth, API Key, Basic Auth, GraphQL) |
| **Copilot Prompt** | Generates a `.copilot-prompt.md` with reference implementations and a skeleton `run.ps1` with TODOs. GitHub Copilot completes the code in VS Code | Complex/custom APIs where you want AI assistance inside the editor |
| **AI/LLM** | Calls OpenAI or GitHub Models API with few-shot examples from existing production functions | Fully automated generation for non-standard patterns |

All 3 modes leverage an **existing function analyzer** that scans production Azure Functions to find similar auth patterns, pagination styles, and output formats — using them as reference examples.

---

## 🛠️ Tech Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | React 18, Vite, WebSocket |
| **Backend** | Node.js, Express, Handlebars |
| **Code Gen** | 4 Handlebars templates + AI-powered generation (OpenAI / GitHub Models) |
| **Schema Detection** | JSON sample parsing → Snowflake type inference + LATERAL FLATTEN |
| **Testing** | Azure Functions Core Tools (`func host start`), automated multi-run harness |
| **Target Runtime** | Azure Functions (PowerShell 7.4), Azure Blob Storage |
| **Data Warehouse** | Snowflake (External Stage → Raw → View → Task) |
| **Monitoring** | Streamlit on Snowflake |

---

## 📊 Pipeline Monitoring Dashboard (Snowflake Streamlit)

After pipelines are deployed, a **Streamlit-on-Snowflake dashboard** provides real-time observability across all 3 environments (DEV → UAT → PRD).

### What It Monitors

| Feature | Description |
|---------|-------------|
| **Cross-Environment View** | See every pipeline's status across DEV, UAT, PRD side-by-side — running, paused, errored, or not yet promoted |
| **Data Freshness** | Tracks when each RAW table was last loaded via `COPY_HISTORY` and `TASK_HISTORY`. Flags stale data (>48h = Stale, >168h = Critical) |
| **Row Count Comparison** | Shows RAW table and INFO view row counts per environment — quickly spot data drift between DEV and PRD |
| **Pipeline Lineage** | Auto-discovers data flow using `SNOWFLAKE.CORE.GET_LINEAGE`: External Stage → RAW table → downstream INFO views |
| **Run Statistics** | Runs, pass/fail counts, average duration per pipeline over configurable lookback (1–30 days) |
| **Data Load Trends** | Daily rows-loaded charts per environment from `COPY_HISTORY`, with fallback to task run counts |
| **Permission Issues** | Dedicated page that detects permission-related failures, extracts role names from errors, and suggests `GRANT` SQL statements |
| **Non-Pipeline Tasks** | Separate view for utility/maintenance tasks not tied to a data pipeline |

![Pipeline Monitoring Dashboard Overview](readme/PipelineMonitoring.png)

### How Pipeline Discovery Works

The dashboard uses an **infrastructure-first approach** — no hardcoded pipeline names:

1. **Stage Scan** — Finds all base tables in `*_STAGE` schemas (these are COPY targets)
2. **Lineage Resolution** — Calls `GET_LINEAGE()` to discover downstream views and tables
3. **Task Matching** — Matches tasks to pipelines by checking if the task's procedure references the source table or any downstream object
4. **Unclaimed tasks** → routed to the Non-Pipeline Tasks page

### Dashboard Pages

| Page | Purpose |
|------|---------|
| 🔄 **Pipeline Monitoring** | Domain-organized pipeline cards with KPIs, lineage, load trends, and table freshness |
| ⚙️ **Non-Pipeline Tasks** | Compact grid of utility tasks with env status dots, schedule, and error info |
| 🔒 **Permission Issues** | Surfaces permission-related failures with auto-generated `GRANT` suggestions |

### Performance

- **10-min SQL cache** — metadata queries run once and are cached
- **Parallel queries** — `ThreadPoolExecutor` (up to 12 workers) for multi-database fetches
- **Session state** — instant page switches with no re-queries
- **Error fallbacks** — every query wrapped in try/except with sidebar notifications

---

## 📁 Project Structure

```
data_pipeline_app/
├── frontend/                     # React + Vite UI
│   └── src/
│       ├── pages/Dashboard.jsx   # 6-tab workflow orchestrator
│       └── components/           # Form wizard, code preview, live console, report
│
├── backend/                      # Express API server
│   ├── server.js                 # HTTP + WebSocket server
│   ├── routes/                   # REST endpoints (generate, test, templates, docs)
│   ├── services/                 # Code gen, schema detection, testing, metrics
│   └── templates/                # Handlebars templates (4 auth patterns)
│
├── azure_function/               # Generated output directory
│   └── <FunctionName>/           # Each generated function
│       ├── run.ps1
│       └── function.json
│
└── readme/                       
```

---

## 🎯 Impact

- **Pipeline creation time**: Hours of manual scripting → minutes of guided configuration
- **3 code generation modes**: Template, Copilot Prompt, AI/LLM — choose your level of automation
- **Automated testing**: No more manual `func host start` + eyeballing logs — structured test runs with pass/fail verdicts
- **Snowflake DDL**: Auto-generated schema detection from sample JSON — including nested array FLATTEN
- **Production monitoring**: Real-time pipeline health across 3 environments with zero manual configuration — dashboard auto-discovers all pipelines from Snowflake metadata
