# Study: Data Refinement (raw table → clean table)

## Concept

This isn't a new concept for you — it's the same ETL discipline as [`Stackoverflow_AI_Trends_V1`](../../../Stackoverflow_AI_Trends_V1/stackoverflow-ai-trends_V1/scripts/etl_pipeline.py), just running **table → table inside Postgres** instead of **CSV → table**. Two ways to do the transform, both worth understanding:

1. **Pandas-side:** `pd.read_sql("SELECT ...", engine)` → clean the DataFrame → `df.to_sql("ai_trends_refined", engine, if_exists="replace")`. Easier to debug interactively (inspect the DataFrame at each step), but round-trips all data through Python.
2. **SQL-side:** `CREATE TABLE ai_trends_refined AS SELECT ... FROM raw_survey_data WHERE ...`. Faster, stays in the database, but harder to debug row-by-row.

For a few thousand survey rows either is fine — pick pandas-side if you want the debugging practice, SQL-side if you want the `CREATE TABLE AS SELECT` (CTAS) pattern practice (it's a very common data-engineering idiom worth knowing).

## Why this step matters for the whole project

An LLM generating SQL is only as good as the schema you show it. `raw_survey_data."AISelect"` with quoted mixed-case columns and undocumented values is a much harder NL2SQL target than `ai_trends_refined.ai_use_status` with a documented value set. This phase is where most of the eventual Phase 4 answer-quality comes from — not the agent code itself.

## Study material

- Pandas → PostgreSQL loading patterns: [ETL Process Using Pandas and SQLAlchemy: A Comprehensive Guide](https://pyquesthub.com/streamlining-data-management-building-an-etl-process-using-pandas-and-sqlalchemy) — covers `to_sql`, connection engines, and the read/transform/write loop.
- `CREATE TABLE ... AS SELECT` (CTAS) — PostgreSQL docs: search "CREATE TABLE AS" in the official Postgres docs for your installed version (18); it's a short page, read it directly rather than a video for this one.
- Refresher on the source project's own column-mapping decisions and the Junior/Senior definition correction: re-read [`Stackoverflow_AI_Trends_V1/.../README.md`](../../../Stackoverflow_AI_Trends_V1/stackoverflow-ai-trends_V1/README.md#L43-L56) before writing the derivation logic — don't re-derive the experience_tier cutoff from scratch, copy the corrected one.

## Notes (fill in as you go)

Went pandas-side: `pd.read_sql` -> clean in a DataFrame -> run the DDL file -> `df.to_sql(..., if_exists="append")`.

**Row-count mismatch chased down:** the plan said "drop rows with NULL ai_use_status/ai_sentiment", but requiring *both* non-null skewed the 2024 adoption rate to 81.8% instead of the known 61.8%. Root cause: in the 2024 survey, everyone who answered `"No, and I don't plan to"` has NULL `AISent` — the sentiment question was skipped for non-adopters, so it's *missing not at random*, not just noise. Dropping on both columns silently deleted ~59k of that bucket and inflated the "Yes" share. Fix: only require `ai_use_status` (AISelect) non-null; leave `ai_sentiment` nullable, and let sentiment-specific queries add their own `WHERE ai_sentiment IS NOT NULL`. That's now documented directly in the table's `COMMENT ON COLUMN` so the NL2SQL agent (and anyone reading the schema) knows not to assume sentiment-completeness.

Also found `AISelect` itself has cross-year schema drift, separate from the already-known `YearsCodePro` one: 2024 uses a flat `'Yes'`, 2025 splits it into `'Yes, I use AI tools daily/weekly/monthly...'`. Normalized all `'Yes, ...'` variants back down to `'Yes'` in `ai_use_status` so the two years stay comparable — that's what made the 61.8% -> 78.5% checkpoint match.

Checkpoint: `SELECT * FROM ai_trends_refined LIMIT 5` returns clean rows; adoption check comes out (2024, 61.8), (2025, 78.5) — matches the source README exactly.
