# Study: FastAPI + PostgreSQL Backend

## Concept

FastAPI's core ideas, in the order you'll meet them building `/ask`:
- **Path operation + Pydantic model:** `@app.post("/ask")` with a request body typed as a Pydantic `BaseModel` (e.g. `class AskRequest(BaseModel): question: str`) — FastAPI validates the incoming JSON against this automatically and gives you a typed Python object, no manual parsing.
- **Dependency injection for the DB session:** the common pattern is a `get_db()` generator function (yields a session, closes it after), injected into route handlers via `Depends(get_db)`. This is the one FastAPI-specific idiom worth actually understanding rather than pattern-matching — it's why FastAPI code always has that `def get_db(): ... yield session` function.
- **SQLAlchemy engine/session vs. Core:** for Phase 3 (fixed queries, no ORM models needed yet), you can use SQLAlchemy Core directly (`engine.connect()`, `conn.execute(text(...))`) rather than the full ORM — simpler, and it's exactly the pattern already in the `postgres-connect` skill's `verify_connection.py`.
- **`StaticFiles`:** `app.mount("/", StaticFiles(directory="static", html=True))` serves `index.html` so the whole app is one FastAPI process, one `uvicorn` command.

## Study material

- **Video:** [FastAPI SQLAlchemy Tutorial 2025 — Build a REST API with SQL](https://www.youtube.com/watch?v=xq1Snezb1rs)
- **Video (Postgres-specific, walks through the full connection setup):** [Create A REST API with FastAPI, SQLAlchemy and PostgreSQL](https://m.youtube.com/watch?v=2g1ZjA6zHRo)
- **Video (deeper on models/relationships, useful once you're comfortable with the basics):** [Python FastAPI Tutorial (Part 5): Adding a Database](https://www.youtube.com/watch?v=NvOV3ig2tGY)
- **Written reference:** [Building a CRUD FastAPI app with SQLAlchemy — Mattermost](https://mattermost.com/blog/building-a-crud-fastapi-app-with-sqlalchemy/) — good for the `get_db()` dependency pattern specifically.

## Where this connects to what you already know

You already ran raw psycopg2/SQLAlchemy connections in `Stackoverflow_AI_Trends_V1/scripts/etl_pipeline.py` and verified connectivity with the `postgres-connect` skill. Phase 3 is the same `create_engine` + `conn.execute(text(...))` pattern, just called from inside a FastAPI route handler instead of a standalone script.

## Notes (fill in as you go)

_Route signatures you ended up with, any dependency-injection gotchas, how you tested the endpoint (curl/Postman/the chat UI itself)._
