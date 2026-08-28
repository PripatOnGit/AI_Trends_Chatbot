# Study: Docker Deployment

## Concept

Two files, two concepts:
- **`Dockerfile`:** packages this app (code + Python deps) into an image — `FROM python:3.13-slim`, copy `requirements.txt`, `pip install`, copy the rest, `CMD` to run uvicorn.
- **`docker-compose.yml`:** runs multiple containers together as one stack — here, the `app` container and a `db` container (official `postgres:18` image), on a shared network, so `app` can reach `db` by service name (e.g. `PGHOST=db` inside the container, vs. `localhost` when running locally).

The one genuinely new idea vs. everything before this phase: **service-name networking**. Inside Docker Compose, containers address each other by service name, not `localhost` — this is the single most common first-time Docker Compose bug (an app that works locally failing in Compose because `PGHOST` is still set to `localhost`).

Seeding the database: either mount an init SQL script (built from Phase 1's `sql/01_create_refined_table.sql` plus an `INSERT`/CTAS of the refined data) into `/docker-entrypoint-initdb.d/` on the `db` container — Postgres images run everything in that directory automatically on first startup — or re-run `scripts/refine_data.py` against the containerized DB once it's up. The init-script approach is more "real world" (reproducible from a clean container every time) and worth doing for the learning value.

## Study material

- **Video (closest match — FastAPI + Postgres + Docker specifically):** [FastAPI with PostgreSQL and Docker](https://www.youtube.com/watch?v=2X8B_X2c27Q)
- **Video (build-along, slightly longer, shows the full compose file being written):** [Build A FastAPI, SQLAlchemy, PostgreSQL, Docker Project With Me #2](https://www.youtube.com/watch?v=NOZzoYqrDLg)
- **Video (recent, Nov 2025 — dev-environment setup with Alembic migrations included, useful if you want migrations too though not required for this project):** [Setting Up a Dev Environment for FastAPI, PostgreSQL, SQLAlchemy & Alembic with Docker](https://www.youtube.com/watch?v=JWQEostzy60)
- **Written reference:** [Dockerizing FastAPI with Postgres, Uvicorn, and Traefik — TestDriven.io](https://testdriven.io/blog/fastapi-docker-traefik/) — skip the Traefik/reverse-proxy part, not needed for local/portfolio deployment; the Postgres + Dockerfile + Compose sections are the relevant ones.

## Notes (fill in as you go)

_Final Dockerfile/compose file shape, any `PGHOST=localhost`-vs-`db` bugs you hit, how you seeded `ai_trends_refined` inside the container._
