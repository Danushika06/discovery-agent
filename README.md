# Discovery Agent

> Agent 3 of the Aivar Innovations AI/ML Hiring Challenge.

Reads unstructured company documents → **discovers the systems** in use (with confidence + evidence) → maps automation use cases and **finds integration gaps** → **generates runnable connector + agent-definition code** for the highest-impact gaps.

Built as a production-grade autonomous agent: it makes context-driven decisions, validates its own output (no hallucinations), fails clearly, stays inside guard rails, and logs every decision to an observable run ledger.

- **Backend:** Python 3.11+, FastAPI, Pydantic v2, typer CLI
- **Frontend:** React + Vite + TypeScript
- **LLMs (free tier):** hybrid **Gemini** (vision + extraction + reasoning) + **Groq/Llama** (fast code generation), behind a swappable provider layer

---

## Quick start

### 1. Get free API keys (no credit card)

- **Gemini:** https://aistudio.google.com/apikey
- **Groq:** https://console.groq.com/keys

### 2. Configure

```bash
cp .env.example .env        # then paste your GEMINI_API_KEY and GROQ_API_KEY
```

### 3a. Run with Docker (one command)

```bash
docker compose up --build
# UI:  http://localhost:5173      API docs: http://localhost:8000/docs
```

### 3b. Run locally

```bash
# Backend
pip install -r backend/requirements.txt
uvicorn backend.app.main:app --reload --port 8000

# Frontend (separate terminal)
cd frontend && npm install && npm run dev
# open http://localhost:5173
```

### 4. Generate the demo knowledge base (optional — already committed)

```bash
python data/generate_kb.py
```

---

## CLI (scriptable, no UI required)

```bash
# Level 1 — discover systems from a folder of mixed documents
python -m backend.cli discover data/kb --out out/inventory.json

# Level 2 — map use cases and prioritize integration gaps
python -m backend.cli gaps --inventory out/inventory.json --use-cases data/use_cases.json

# Level 3 — generate connector + agent code for the top gaps
python -m backend.cli codegen --gaps out/gaps.json --top-n 3
# -> writes out/connectors_<run>.zip with validated, runnable connectors
```

The CLI prints rich summaries; full structured JSON + Markdown reports are written to `out/`.

---

## What each level does

| Level | Input | Output | Key guarantee |
|---|---|---|---|
| **1 — Discovery** | PDF, image, spreadsheet, Markdown, text | System inventory: name, category, auth, entities, processes, criticality, **confidence**, **source evidence** | **No hallucinations** — every claim is grounded in source text; uncertain findings flagged for review |
| **2 — Gap analysis** | Inventory + automation use cases | Use-case → system mappings, data flows, **prioritized integration gaps**, dependency statements | Priority is computed deterministically (frequency × criticality × downstream blocked) — explainable |
| **3 — Code generation** | Gaps | Per gap: connector (auth, CRUD, pagination, rate limiting, retries, errors) + agent-definition YAML + unit tests + README, as a zip | Generated code is **validated** (syntax + import + YAML + a real pytest run) before download |

---

## Tests

```bash
pip install -r backend/requirements.txt
python -m pytest backend/tests -q
```

The suite runs **fully offline** (fake LLM router) and covers: multi-format ingestion, the
hallucination/grounding guard, the confidence policy, gap reconciliation + prioritization,
and code generation across every auth/pagination branch — including **running the generated
connectors' own pytest suites** to prove they work.

---

## Configuration (`.env`)

| Variable | Purpose | Default |
|---|---|---|
| `GEMINI_API_KEY`, `GROQ_API_KEY` | Free-tier keys | — |
| `GEMINI_MODEL`, `GROQ_CODE_MODEL` | Override models (free-tier ids rotate) | `gemini-2.5-flash`, `llama-3.3-70b-versatile` |
| `ROUTE_EXTRACTION` / `ROUTE_VISION` / `ROUTE_GAP_ANALYSIS` / `ROUTE_CODEGEN` | Which provider handles each task | gemini / gemini / gemini / groq |
| `CONFIDENCE_EXPLICIT` / `CONFIDENCE_INFERRED` | Confidence-tier thresholds | 0.95 / 0.70 |
| `GEMINI_RPM`, `GROQ_RPM` | Free-tier rate limits | 10 / 30 |

## Image documents & OCR

Image documents (screenshots/scans) are read **vision-first** via Gemini. A local
**Tesseract OCR** fallback provides a text layer for self-grounding. Tesseract is optional —
without it, image-only systems are still discovered via vision and grounded if they also
appear in a text document (see the Dockerfile to enable Tesseract in the container).

## Project layout

See `CLAUDE.md` for the full architecture map and conventions. Key directories:
`backend/app/llm` (provider layer), `discovery` (L1), `gapanalysis` (L2), `codegen` (L3),
`routers` (API); `frontend/src` (React UI); `data/` (sample knowledge base).
