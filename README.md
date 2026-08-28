# AI Trends Chatbot

An end-to-end natural-language chatbot over the **AI Adoption & Sentiment Trends** dataset (Stack Overflow Developer Survey 2024-2025, from [`../Stackoverflow_AI_Trends_V1/`](../Stackoverflow_AI_Trends_V1/stackoverflow-ai-trends_V1/)). The user asks a question in a chat window; an agent turns it into SQL, runs it against a refined PostgreSQL table, and answers using the results.

Built as a learning-by-doing project: each build phase pairs a development task with study material, so the concepts are understood, not just copy-pasted. See [PLAN.md](PLAN.md) for the phased plan and [docs/study_plan/](docs/study_plan/) for per-phase notes and resources.

## Stack

- **Data:** PostgreSQL 18 (local), refined from the existing `raw_survey_data` table
- **Backend:** FastAPI
- **Agent:** OpenAI Agents SDK (SQL-generation tool + execution tool)
- **Frontend:** plain HTML/CSS/JS chat window (no framework, to keep the FastAPI/agent layer the learning focus)
- **Deployment:** local first, then Docker Compose (app + Postgres)

## Environment

Project-level venv (not the workspace-wide `Stackoverflow_AI_Trends_V1/.venv` — this is a separate deployable app):

```
python -m venv .venv
./.venv/Scripts/pip install -r requirements.txt
```

Copy `.env.example` to `.env` and fill in `OPENAI_API_KEY` and DB credentials before running anything that talks to OpenAI or Postgres.

## Status

See [PLAN.md](PLAN.md) — Phase 1 (data refinement) not yet started as of 2026-08-26.
