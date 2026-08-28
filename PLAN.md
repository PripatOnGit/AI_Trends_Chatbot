# Build Plan: AI Trends Chatbot

Learning-with-development plan. Each phase = one concept studied + one working piece shipped, in order. Don't start a phase's dev task until the study material for it has been at least skimmed — the point is understanding what the code does, not just running it.

Environment: project-level `.venv` in this folder (see [README.md](README.md)). Source data: `raw_survey_data` table in the `ai_sentiment_db` Postgres database, populated by [`../Stackoverflow_AI_Trends_V1/stackoverflow-ai-trends_V1/scripts/etl_pipeline.py`](../Stackoverflow_AI_Trends_V1/stackoverflow-ai-trends_V1/scripts/etl_pipeline.py) — that table stays read-only from this project; we build a new table from it.

## Phase 1 — Refine `raw_survey_data` into a clean AI Trends table

**Goal:** a small, well-typed, well-named table this project owns — `ai_trends_refined` — that an LLM can generate accurate SQL against. Raw survey tables are bad NL2SQL targets: cryptic PascalCase column names (`AISelect`, `AISent`, `YearsCodePro`), free-text-ish values, nulls. Refining first is what makes Phase 4 (the agent) actually work well later.

**What "refined" means here, concretely:**
- Select and rename columns to plain snake_case: `survey_year`, `ai_use_status` (from `"AISelect"`), `ai_sentiment` (from `"AISent"`), `years_experience` (from `"YearsCodePro"` — see the workbook's fallback-to-`YearsCode` note if the value is missing).
- Derive `experience_tier` (`'Junior'` if `years_experience <= 3` else `'Senior'`) as a real column instead of recomputing it in every query — matches the definition already used in the source project's Query 3, not the older `>= 8 years` definition mentioned in that README as superseded.
- Drop rows with NULL `ai_use_status`/`ai_sentiment` (mirrors the `WHERE ... IS NOT NULL` pattern already used in the source project's queries).
- Add a short `column_glossary` reference (a markdown table or a Postgres `COMMENT ON COLUMN`) — this becomes the schema description you hand the agent in Phase 4 so it doesn't have to guess what `ai_sentiment` values mean.

**Dev task:**
1. `sql/01_create_refined_table.sql` — `CREATE TABLE ai_trends_refined (...)`.
2. `scripts/refine_data.py` — reads `raw_survey_data` via SQLAlchemy, applies the renames/derivation/filtering (pandas or plain SQL `INSERT INTO ... SELECT`, your call), writes into `ai_trends_refined`.
3. Validate: row counts before/after, `SELECT DISTINCT experience_tier, ai_use_status` sanity check, compare one aggregate (e.g. YoY adoption %) against the known result in the source README (61.8%→78.5%) to confirm the refined table agrees with the original analysis.

**Study material:** [docs/study_plan/01_data_refinement.md](docs/study_plan/01_data_refinement.md)

**Checkpoint:** `SELECT * FROM ai_trends_refined LIMIT 5;` returns clean rows; the YoY adoption check matches the known 61.8%→78.5% figure.

---

## Phase 2 — Chat HTML interface

**Goal:** a static, framework-free chat window: message list, input box, send button, calls a backend endpoint, renders the reply. No agent or real backend logic yet — this phase is pure frontend, and can be built/tested against a fake/hardcoded response before Phase 3 exists.

**Dev task:** `static/index.html` (+ inline or separate CSS/JS) with:
- A scrollable message list (user messages right-aligned, bot messages left-aligned, is the common pattern).
- An input box + submit that POSTs `{ "question": "..." }` via `fetch()` to `/ask` and appends the JSON response to the message list.
- Basic loading/error states (disable input while waiting, show an error bubble on failure).

**Study material:** [docs/study_plan/02_chat_ui.md](docs/study_plan/02_chat_ui.md)

**Checkpoint:** opening `static/index.html` directly in a browser (file://) lets you type a message and see it appended to the thread, even with a stubbed `fetch` that returns a canned string — proves the UI loop works before it's wired to anything real.

---

## Phase 3 — FastAPI app: run a query, return results

**Goal:** a real backend the chat window can talk to. At this stage the endpoint executes a **fixed/parameterized SQL query** against `ai_trends_refined` — no LLM yet — to prove the FastAPI-to-Postgres path end to end before adding agent complexity on top.

**Dev task:** `app/main.py`:
- `POST /ask` — Pydantic request model `{question: str}`, for now maps the question to one of a few hardcoded queries (e.g. keyword match: "adoption" → adoption-rate query, "sentiment" → sentiment query) or just runs one fixed query, and returns the result rows as JSON.
- Reuse the SQLAlchemy connection pattern already used elsewhere in this workspace (the user-level `postgres-connect` skill has the URL-building snippet).
- Serve `static/index.html` via FastAPI's `StaticFiles` so the whole app is one process.
- CORS: not needed yet since frontend is served by the same FastAPI app.

**Study material:** [docs/study_plan/03_fastapi_backend.md](docs/study_plan/03_fastapi_backend.md)

**Checkpoint:** `uvicorn app.main:app --reload`, open `http://localhost:8000`, type a question, get back real rows from `ai_trends_refined` rendered in the chat window.

---

## Phase 4 — OpenAI Agents SDK: NL2SQL agent

**Goal:** replace the hardcoded query mapping from Phase 3 with an agent that (1) generates SQL from the natural-language question, (2) executes it against `ai_trends_refined`, (3) answers the question using the result rows. This is the core "learning" phase — text-to-SQL agents are a distinct pattern from the RAG/embedding chatbot design considered earlier for this same dataset; no pgvector/embeddings needed here, since the answer comes from live SQL results, not semantic retrieval over precomputed findings.

**Dev task:** `app/agent.py`:
- One tool, `run_sql_query(query: str) -> str`, that executes against Postgres (read-only role or `SELECT`-only validation — don't let the agent run arbitrary DDL/DML against this table) and returns rows as text/JSON.
- Agent instructions include the refined table's schema + `column_glossary` from Phase 1 (so the model knows `ai_use_status` values are e.g. `'Yes'`/`'No'`, not guessing).
- `Runner.run(agent, question)` wired into `POST /ask` from Phase 3, replacing the keyword-matching stopgap.
- Guardrail: cap result rows returned to the model (e.g. `LIMIT 50`) so a broad query doesn't blow the context or the answer quality.

**Study material:** [docs/study_plan/04_nl2sql_agent.md](docs/study_plan/04_nl2sql_agent.md)

**Checkpoint:** ask "What was the YoY change in AI adoption?" and "How does the junior/senior adoption gap compare across years?" in the chat window and get correct, data-grounded answers (matching the known README figures) generated via live SQL, not a hardcoded query.

---

## Phase 5 — Docker deployment

**Goal:** the whole stack (`app` + `postgres`) runs with one `docker-compose up`, matching how the interview story should end ("built locally, then containerized").

**Dev task:**
- `docker/Dockerfile` — `FROM python:3.13-slim`, copy `requirements.txt`/app code, `CMD uvicorn app.main:app --host 0.0.0.0`.
- `docker/docker-compose.yml` — two services: `app` (this project) and `db` (official `postgres:18` image, seeded with `ai_trends_refined` — either via an init SQL script mounted into `/docker-entrypoint-initdb.d/` or by re-running Phase 1's refine script against the containerized DB).
- `.env` read by both `docker-compose.yml` and the app (don't bake credentials into the image).

**Study material:** [docs/study_plan/05_docker_deployment.md](docs/study_plan/05_docker_deployment.md)

**Checkpoint:** `docker compose up --build` from a clean checkout brings up a working chatbot with no manually-run local setup steps.

---

## After all 5 phases

Optional stretch (only if there's time before it needs to be interview-ready, per the existing prep notes): deploy to AWS (RDS for Postgres + App Runner/Elastic Beanstalk for the app) — same as the stretch goal already recorded for this project. Not required for the core deliverable.
