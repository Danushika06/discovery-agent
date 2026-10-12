# Discovery Agent — Systems Discovery, Gap Analysis and Connector Code Generation

**A three-stage LLM agent that reads a folder of unstructured company documents, builds a grounded inventory of the software systems in use, finds the missing integrations between them, and generates validated connector code for the highest-impact gaps.**

![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white) ![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat&logo=fastapi&logoColor=white) ![Pydantic](https://img.shields.io/badge/Pydantic-E92063?style=flat&logo=pydantic&logoColor=white) ![Gemini](https://img.shields.io/badge/Gemini-8E75B2?style=flat&logo=googlegemini&logoColor=white) ![Groq](https://img.shields.io/badge/Groq-F55036?style=flat) ![Jinja](https://img.shields.io/badge/Jinja-B41717?style=flat&logo=jinja&logoColor=white) ![React](https://img.shields.io/badge/React-20232A?style=flat&logo=react&logoColor=61DAFB) ![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white) ![Vite](https://img.shields.io/badge/Vite-646CFF?style=flat&logo=vite&logoColor=white) ![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white) ![pytest](https://img.shields.io/badge/pytest-0A9EDC?style=flat&logo=pytest&logoColor=white)

## Overview

Before a company can automate a workflow such as "when a deal closes in the CRM, create the invoice in the ERP", someone has to work out which systems exist, which of them already talk to each other, and which integrations are missing. That information sits in architecture PDFs, spreadsheets, wiki pages, email threads and screenshots. Asking an LLM to summarise those files is easy; getting a result you can trust is harder, because models invent plausible product names.

This project is an agent that does that work in three levels and is built around not trusting the model. The repo describes itself as Agent 3 of a hiring challenge from Aivar Innovations. Level 1 discovers systems and checks every claim against the source text. Level 2 maps automation use cases onto those systems and ranks the integration gaps with a formula in code instead of asking the model for a ranking. Level 3 generates connector code from Jinja templates and then actually imports it and runs its tests before offering it for download.

Status: all three levels, the CLI, the API and the React UI are implemented, and the backend has offline tests that use a fake LLM router. Quality of real output depends on the Gemini and Groq free-tier models; I have not recorded accuracy numbers, so this README makes no claims about them. The sample knowledge base in `data/kb` is synthetic demo data for a made-up company.

## Key Features

- **Mixed-format ingestion** — `ingest/loaders.py` dispatches PDFs (pdfplumber, with `[page N]` markers and tables), images, spreadsheets (pandas/openpyxl, cell references like `Integrations!A2`), Markdown and text. Unsupported or corrupt files are recorded as `SkippedDocument` with a `needs` message, never silently dropped.
- **Vision-first image reading** — images go to Gemini as image parts; an optional Tesseract OCR pass adds a text layer so the result can be grounded.
- **Hallucination guard** — each evidence quote must appear in its source document (`discovery/grounding.py`: normalised substring or at least 60% token overlap). An ungrounded system that has a text source gets its confidence capped at 0.49 and is flagged for review.
- **Confidence tiers** — `explicit` (at least 0.95), `inferred` (at least 0.70) and `review` (below that), with thresholds in `.env`.
- **Cross-document merge** — mentions of the same system are merged by a normalised name key; entities and processes are unioned, the highest confidence and criticality win, and each mention keeps its own evidence record.
- **Deterministic gap ranking** — `gapanalysis/prioritizer.py` computes a business impact score from use-case frequency, criticality and the number of use cases a gap blocks, then buckets priorities relative to the run's maximum.
- **Template-based code generation** — the model only fills a `ConnectorSpec` (auth type, entities, pagination style); the connector, its tests and README come from Jinja2 templates, so structure is not left to the model. An `agent.yaml` is assembled with `yaml.safe_dump`.
- **Validation before download** — `codegen/validators.py` runs `ast.parse`, an import smoke test, a YAML parse and the generated `pytest` suite in a temp directory; results are shown per artifact.
- **Provider layer** — `LLMRouter` maps each task (`extraction`, `vision`, `gap_analysis`, `codegen`) to Gemini or Groq via env vars; providers are built lazily so a missing key only fails the task that needs it. A per-provider rate limiter and a retry wrapper with exponential backoff sit in `llm/base.py`.
- **Run ledger** — every decision, LLM call, confidence score and failure is written as a JSON line to `data/runs/<run_id>.jsonl` and exposed through `/runs`.
- **Three front doors** — a typer CLI, a FastAPI service and a React UI that chains the three steps and shows the ledger in a side panel.

## How It Works

```
documents --> ingest --> extract (per doc) --> merge --> ground + confidence policy --> Inventory
Inventory + use cases --> LLM mapping --> reconcile --> deterministic prioritise --> GapReport
top-N missing gaps --> ConnectorSpec (LLM) --> Jinja render --> agent.yaml --> validate --> zip bundle
```

### Level 1: discovery (`backend/app/discovery/`)

`service.run_discovery()` orchestrates the run:

1. `ingest.loaders.discover_documents()` finds supported files; `load_document()` returns an `ExtractedDoc` with `text`, optional `images`, the ingest method and warnings.
2. `extractor.extract_from_doc()` picks a path per document: `vision` if it has an image, `extraction` if it has text. The system prompt forbids inventing systems or guessing a product from a generic description (an unnamed "marketing automation platform" must stay generic with low confidence), and requires `evidence_quote` to be copied verbatim. The result is a `DocumentExtraction` of `ExtractedSystem` records.
3. `merger.merge_systems()` groups mentions across documents.
4. `_apply_grounding_and_policy()` checks each evidence quote with `grounding.is_grounded()`. If nothing is grounded and a text source exists, confidence is capped at 0.49 (`_HALLUCINATION_CAP`) and a note is added. If the only source is an image with no text layer, it adds a note that the system could not be self-verified. Then `_assign_tier()` sets the tier and `needs_review`.
5. The result is an `Inventory` with systems, warnings and skipped documents.

### Level 2: gap analysis (`backend/app/gapanalysis/`)

`mapper.analyze()` gives the model the inventory summary and the use cases from `data/use_cases.json` and asks for use-case-to-system mappings, data flows and the integrations required, marking one `available` only if the inventory evidence says it exists. `gap_detector.reconcile()` then turns that into domain objects, flags required systems that are absent from the inventory, and assigns ids (`gap-1`, ...). `prioritizer.prioritize()` computes the score:

```
business_impact_score = max over requiring use cases of (frequency / max_frequency * criticality_weight)
                        * (1 + 0.5 * (blocks_count - 1))
```

Criticality weights are 4, 3, 2, 1. A gap that already exists is forced to low priority, the rest are bucketed at 75%, 50% and 25% of the highest score, and dependency statements such as "Integration X must exist before use case Y can be automated" are generated.

### Level 3: code generation (`backend/app/codegen/`)

`service.generate_code()` takes the top N missing gaps by impact and, for each one: `connector_gen.generate_connector_spec()` asks for a `ConnectorSpec` for the destination system and normalises names; `render_connector_files()` renders `connector.py`, `test_connector.py`, `README.md` and `requirements.txt` from `templates/*.j2` (the connector template has branches for OAuth2 client credentials, API key bearer, basic and custom header auth, and cursor, offset and page pagination, plus rate limiting, retries and typed errors); `agentdef_gen.generate_agent_def()` writes `agent.yaml`; `validators.validate_artifact()` checks it. A failure in one gap is recorded and does not stop the others. Results are written to `out/connectors_<run_id>.zip`, with each gap in its own folder.

### Provider layer and observability

`llm/base.py` defines `BaseLLMProvider`, which handles rate limiting, retries and validation of the JSON response against the Pydantic schema. Gemini and Groq both get the schema injected into the system prompt and JSON mode; the Gemini provider comments that it avoids `response_schema` because Gemini rejects schema defaults. Errors are typed (`MissingAPIKeyError`, `LLMRateLimitError`, `LLMResponseError`, `ProviderUnavailableError`), each carrying a `needs` message that the API returns as a 502 detail and the CLI prints. `observability.RunLedger` is passed through every service.

### Frontend

`frontend/src/App.tsx` holds the state for the three steps (`DiscoveryStep`, `GapStep`, `CodegenStep`) and an `Observability` panel that loads a run's ledger. `api.ts` uses `VITE_API_BASE` if set, otherwise `/api`, which the Vite dev server proxies to `localhost:8000`.

## Tests

`backend/tests/` has three offline test files. `test_discovery_offline.py` uses a `FakeRouter` that returns grounded extractions and also injects a deliberately hallucinated system ("Initech Mainframe") to check the guard. `test_gaps_offline.py` checks reconciliation, scoring and ordering. `test_codegen_offline.py` renders every auth and pagination combination, runs a generated connector's own pytest suite, and runs the end-to-end bundle generation. No API keys are needed.

## Tech Stack

- **Backend:** Python (Dockerfile uses 3.12), FastAPI, Uvicorn, Pydantic v2 and pydantic-settings, typer, rich, tenacity
- **LLMs:** Google Gemini via `google-genai` (default `gemini-2.5-flash`), Groq via `groq` (default `llama-3.3-70b-versatile`)
- **Ingestion:** pdfplumber, pypdf, Pillow, pytesseract (optional), pandas, openpyxl
- **Code generation:** Jinja2, PyYAML
- **Frontend:** React 18, TypeScript, Vite
- **Infra and tests:** Docker, Docker Compose, pytest

## Getting Started

### Prerequisites

- Python 3.11 or newer, Node.js 20 (the frontend image uses `node:20-alpine`)
- Free API keys for Gemini (https://aistudio.google.com/apikey) and Groq (https://console.groq.com/keys)
- Optional: Tesseract OCR installed locally for image text grounding (a commented-out line in `backend/Dockerfile` enables it in the container)

### Installation

```bash
git clone https://github.com/Danushika06/discovery-agent.git
cd discovery-agent
cp .env.example .env      # then add GEMINI_API_KEY and GROQ_API_KEY
pip install -r backend/requirements.txt
```

### Running

Backend and frontend locally:

```bash
uvicorn backend.app.main:app --reload --port 8000

# in a second terminal
cd frontend && npm install && npm run dev
# open http://localhost:5173
```

Docker:

```bash
docker compose up --build
# UI: http://localhost:5173   API docs: http://localhost:8000/docs
```

Note: `docker-compose.yml` passes a fixed `VITE_API_BASE` (a remote IP address) to the frontend build, so the containerised UI will call that address instead of your local backend unless you change it to `http://localhost:8000`.

CLI, one command per level:

```bash
python -m backend.cli discover data/kb --out out/inventory.json
python -m backend.cli gaps --inventory out/inventory.json --use-cases data/use_cases.json
python -m backend.cli codegen --gaps out/gaps.json --top-n 3
```

Tests:

```bash
python -m pytest backend/tests -q
```

The sample knowledge base in `data/kb` is already committed; `python data/generate_kb.py` rebuilds it.

### Configuration

All settings are read from `.env` (see `.env.example`): `GEMINI_API_KEY`, `GROQ_API_KEY`, model ids (`GEMINI_MODEL`, `GEMINI_VISION_MODEL`, `GROQ_CODE_MODEL`, `GROQ_FAST_MODEL`), task routing (`ROUTE_EXTRACTION`, `ROUTE_VISION`, `ROUTE_GAP_ANALYSIS`, `ROUTE_CODEGEN`), `CONFIDENCE_EXPLICIT` / `CONFIDENCE_INFERRED`, `GEMINI_RPM` / `GROQ_RPM`, `LLM_MAX_RETRIES`, `STRICT_INGEST` and `RUN_LEDGER_DIR`.

## Usage

| Endpoint | Method | What it does |
| --- | --- | --- |
| `/health` | GET | Which providers have keys, active models and task routes |
| `/discovery/run` | POST | Level 1. Multipart `files` upload or a form `path`; returns inventory plus a Markdown report and writes `out/inventory.json` |
| `/gaps/run` | POST | Level 2. JSON body with `inventory` and optional `use_cases` or `use_cases_path` (defaults to `data/use_cases.json`); writes `out/gaps.json` |
| `/codegen/run` | POST | Level 3. Body with `gaps` or `report`, `top_n` (default 3), `run_tests`; returns artifacts and a `bundle_url` |
| `/codegen/bundle/{run_id}` | GET | Download the generated zip |
| `/runs` | GET | Recent runs, newest first (`limit` query parameter) |
| `/runs/{run_id}` | GET | The full event ledger for a run |

LLM failures come back as a `502` whose detail has `error`, `needs`, `provider` and `model`. CORS is open to all origins in `main.py`, which the code marks as demo-only.

## Project Structure

```
discovery-agent/
├── backend/
│   ├── cli.py                    # typer CLI: discover | gaps | codegen
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── app/
│   │   ├── main.py               # FastAPI app, CORS, routers
│   │   ├── config.py             # Settings: keys, models, routing, thresholds
│   │   ├── observability.py      # RunLedger, JSONL ledger, list/load runs
│   │   ├── ingest/               # pdf, image, spreadsheet, text loaders + dispatcher
│   │   ├── discovery/            # extractor, grounding, merger, service (Level 1)
│   │   ├── gapanalysis/          # mapper, gap_detector, prioritizer, service (Level 2)
│   │   ├── codegen/              # connector_gen, agentdef_gen, validators, service, templates/*.j2 (Level 3)
│   │   ├── llm/                  # base provider, Gemini, Groq, router, typed errors
│   │   ├── routers/              # health, discovery, gaps, codegen, runs
│   │   └── schemas/              # inventory, gaps, artifacts (Pydantic)
│   └── tests/                    # offline tests with a fake LLM router
├── frontend/                     # React + Vite + TypeScript UI (steps + observability panel)
├── data/
│   ├── kb/                       # synthetic sample knowledge base (PDF, PNG, XLSX, MD, TXT)
│   ├── generate_kb.py            # rebuilds the sample knowledge base
│   └── use_cases.json            # six sample automation use cases
├── docker-compose.yml
└── .env.example
```

---

## Contact

<table>
  <tr><td><b>GitHub:</b> <a href="https://github.com/Danushika06">Danushika06</a></td><td><b>LinkedIn:</b> <a href="https://www.linkedin.com/in/danushika-n-323b4628b/">danushika-n</a></td></tr>
  <tr><td colspan="2" align="center"><b>Email:</b> <a href="mailto:danushikanagarajan@gmail.com">danushikanagarajan@gmail.com</a></td></tr>
</table>

Built by **Danushika N**
