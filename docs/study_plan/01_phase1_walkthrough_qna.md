# Phase 1 Walkthrough: Data Refinement (raw_survey_data -> ai_trends_refined)

Guided, step-by-step build log with the actual Q&A from the session, the learning gaps it surfaced, and likely interview questions this phase maps to. Companion to [01_data_refinement.md](01_data_refinement.md) (the original study material) — this file is the record of *how the learning actually went*, not the plan.

---

## Step 1 — Design the target schema (`sql/01_create_refined_table.sql`)

**Concept:** An NL2SQL agent's answer quality depends on how clean the schema it's shown is. Two known issues in the source data shaped the design:
- `AISelect` (adoption answer) has cross-year drift — 2024 is a flat `'Yes'`, 2025 splits it into `'Yes, I use AI tools daily/weekly/monthly'`.
- `AISent` (sentiment) is `NULL` for most of 2024's `"No, and I don't plan to"` respondents — the question was skipped for non-adopters (missing *not at random*, not just noise).

**Built:**
```sql
CREATE TABLE ai_trends_refined (
    survey_year       INTEGER NOT NULL,
    ai_use_status     TEXT NOT NULL,
    ai_sentiment      TEXT,
    years_experience  NUMERIC,
    experience_tier   TEXT
);
```

**Q&A:**

Q: Why is `ai_use_status` `NOT NULL` but `ai_sentiment` nullable?
A (first pass): "developer needs to select if they use AI tools or not; if not, sentiment stays null."
Correction: the real reason is about what *this table's own build script guarantees*, not survey behavior — the refine script filters out any row with a null `AISelect` before inserting, so `NOT NULL` documents that invariant. `ai_sentiment` stays nullable because we deliberately do **not** filter on it (dropping those rows would have skewed the 2024 adoption rate from 61.8% to 81.8% — see Step 3).

Q: Why is `years_experience` `NUMERIC` and not `INTEGER`?
A (first pass): "survey options for this question were alphanumeric responses."
Correction: by the time data reaches `raw_survey_data`, the upstream ETL (`Stackoverflow_AI_Trends_V1/scripts/etl_pipeline.py`) has already converted text answers ("Less than 1 year" -> `0`, "More than 50 years" -> `51`) into `double precision`. `NUMERIC` here mirrors that already-numeric source type — the original survey text conversion happened earlier in the pipeline, not at this step.

Q: Where should the Junior/Senior derivation logic (`experience_tier`) live — DDL or the Python script?
A: Python script — correct on first answer. DDL just declares structure; conditional/derived logic needs an actual expression language.

**Learning gap identified:** reasoning about `NOT NULL`/type choices from "what the survey looks like" instead of "what my own pipeline enforces." Worth remembering for any schema-design question — constraints should reflect *guarantees your code makes*, not assumptions about upstream data.

---

## Step 2 — Read the raw data

**Concept:** Pull only the needed columns via SQLAlchemy engine + `pd.read_sql` before doing any cleaning.

**Detour — environment bug (not a concept gap, but real debugging practice):**
- First error: `ModuleNotFoundError: No module named 'dotenv'` — running the script with `python` resolved to a *different* project's venv (`Stackoverflow_AI_Trends_V1/.venv`), not this project's.
- Root cause understood: **venv activation is a shell-session setting, not a per-directory one.** `cd`-ing into a different project does not deactivate/reactivate anything; whatever venv was activated earlier in that terminal session stays active.
- Fix: `deactivate` then activate the right one — command differs by shell (`Activate.ps1` for PowerShell, `activate.bat` for cmd, `source .../activate` for Git Bash).

**Two code bugs caught and fixed:**
1. `print(f"Read {raw.head} rows...")` — `raw.head` (no parens) is a method reference, not a row count.
2. `load_dotenv` import/call left commented out, silently "working" only because hardcoded fallback defaults (`os.getenv("PGHOST", "localhost")`, etc.) happened to match `.env`. Flagged as fragile: `OPENAI_API_KEY` (added in a later phase) has no safe hardcoded default, so this habit would break there.

**Result:** 360,130 rows read — matches `raw_survey_data`'s total (261,748 [2024] + 98,382 [2025]).

**Learning gap identified:** relying on coincidental hardcoded defaults instead of actually loading `.env`. Not caught by "it runs fine" — only caught by reasoning about a future case (the API key) where there's no safe default to fall back to.

---

## Step 3 — Clean & derive columns

**Concept:** Filter only on `AISelect` (not `AISent`), normalize `AISelect`'s cross-year drift, derive `experience_tier` with explicit NaN-handling.

**Q&A:**

Q: What does `.apply()` do — called once for the whole column, or once per row? Why can't you write `clean["AISelect"].startswith("Yes")` directly?
A: "apply() is used if we want to apply any function to df" (partial).
Filled in: `.apply()` calls the function once per value/row — pandas loops element-by-element, unlike a vectorized operation that acts on the whole array without a per-element Python loop. `.startswith()` is a plain `str` method; a Series doesn't have it directly — the vectorized equivalent is `.str.startswith(...)` via pandas' `.str` accessor (faster, no per-row loop) — a cleaner alternative to `.apply()` here, noted but not required for this build.

Q: `NaN <= 3` — True, False, or something else? Which tier does a row with unknown experience silently get assigned if you don't special-case it?
A: (asked directly for the answer) — any comparison involving `NaN` is `False` (IEEE-754 float behavior). The naive `"Junior" if years_experience <= 3 else "Senior"` therefore silently mislabels every unknown-experience respondent as `"Senior"`. Fix: `None if pd.isna(y) else (...)`.

Q: The printed `experience_tier` column shows `NaN`, even though the lambda returns `None` for missing years. Display quirk, or did something convert it?
A (first pass): "Null in raw file got converted in NaN by SQL."
Correction: unrelated to SQL — verified directly (`type(result.iloc[1])` -> `<class 'float'>`, value `nan`) that **pandas itself silently normalizes a returned `None` back into float `NaN`** when building the resulting Series, because pandas' string/object dtype uses `NaN` as its standard missing-value marker. Functionally harmless here (`to_sql()` converts `NaN` -> SQL `NULL` on write either way), but the practical lesson: don't rely on `is None` checks against a pandas column — use `pd.isna()`, which catches both.

**Learning gap identified:** attributing a pandas-internal behavior (None -> NaN coercion) to "SQL" without verifying — good instinct to look for *a* cause, but the cause was misattributed to the wrong layer of the stack (Python/pandas vs. the database). General pattern to watch for: when something unexpected shows up in output, isolate *which layer* (pandas vs. SQL vs. driver) actually produced it before explaining it.

---

## Step 4 — Write to Postgres

**Built:** run the DDL file via `engine.begin()` + `conn.execute(text(ddl))`, then `df.to_sql("ai_trends_refined", engine, if_exists="append", index=False)`.

**Anomaly:** first reported row count (322,068) didn't match the expected 311,068 (360,130 total − 49,062 dropped for null `AISelect`). Re-checking the DB directly showed the table already at the correct 311,068 with the correct per-category breakdown — likely a stale/leftover table state from before the "start from scratch" reset, resolved by re-running cleanly. Not fully root-caused, but the final DB state was independently verified correct.

**Practice note (not a concept gap):** part of Step 3 and the `validate()` step in Step 5 were copy-pasted from the pre-written reference file (`refine_data.reference.py`) rather than typed independently. Flagged in-session; worth self-monitoring on future phases since the stated goal was learning-by-typing, not learning-by-reading.

---

## Step 5 — Validate against the known checkpoint

**Query:**
```sql
SELECT survey_year,
       ROUND(100.0 * SUM(CASE WHEN ai_use_status = 'Yes' THEN 1 ELSE 0 END) / COUNT(*), 1) AS pct_yes
FROM ai_trends_refined
GROUP BY survey_year
ORDER BY survey_year;
```
**Result:** `(2024, 61.8)`, `(2025, 78.5)` — matches the source README's known adoption figures exactly.

**Q&A:**

Q: Why `100.0 *` instead of `100 *`?
A (first pass): "it will round to nearest value with .0, not with actual float value."
Correction: not about rounding precision — it's about **integer division truncating before `ROUND` ever runs**. `SUM(...)` and `COUNT(*)` are both integers; `100 * SUM(...) / COUNT(*)` would be evaluated in integer arithmetic, discarding everything after the decimal point *before* `ROUND` gets a chance to act on it (e.g. `0/1` style truncation, not `0.06...`). `100.0 *` forces `numeric` arithmetic for the whole expression, so the division carries real decimal precision.

Q: Shorter equivalent of `SUM(CASE WHEN...)`? And does the unfiltered `COUNT(*)` denominator count all rows or only `'Yes'` rows?
A: `COUNT(*) FILTER (WHERE ai_use_status = 'Yes')` for the numerator (correct on first answer). Denominator: "it will count all rows (yes+no)" (correct on second try, after a first partial answer).

**Learning gap identified:** attributing SQL integer-division truncation to "rounding precision" rather than "wrong arithmetic type before rounding even happens." This is a common SQL gotcha worth internalizing since it's silent (no error — just a wrong, often-zero, number).

---

## Learning gaps summary (for review before an interview)

1. **Constraint reasoning:** justify `NOT NULL`/nullability from what your *own pipeline* guarantees, not assumptions about the source survey.
2. **Trace column types to their actual origin:** check what the *immediate* source table's type already is (`raw_survey_data.YearsCodePro` was already `double precision`) rather than reasoning from the original raw CSV format.
3. **`NaN` comparison semantics:** any comparison against `NaN` is `False` — an un-special-cased `<=`/`>=`/`==` against missing numeric data silently produces a specific, wrong, non-obvious branch instead of erroring.
4. **`None` vs `NaN` in pandas:** a function returning `None` inside `.apply()` does not necessarily survive as `None` in the resulting Series — pandas normalizes it to `NaN` for object/string dtype columns. Use `pd.isna()`, not `is None`, when checking pandas values.
5. **Root-causing across layers:** when output looks wrong, identify which layer (Python/pandas vs. SQL/Postgres vs. the driver) actually produced the behavior before explaining it — don't default to blaming "the database" for something pandas did.
6. **SQL integer division:** `int / int` truncates before any `ROUND()`/formatting happens; multiplying by `100.0` (not `100`) forces `numeric` arithmetic early enough to preserve precision.
7. **Environment/venv scoping:** venv activation is tied to the *shell session*, not the current directory — switching project folders in an already-activated terminal does not switch environments.

---

## Likely interview questions this phase maps to

**Data cleaning / pandas:**
- "How does pandas represent missing data, and what's the difference between `None`, `NaN`, and SQL `NULL` once you're moving data between Python and a database?"
- "What's the difference between `.apply()` and a vectorized pandas operation? When would you avoid `.apply()`?"
- "Why can naive comparisons against missing numeric data (e.g. `df['x'] <= 3`) introduce silent bugs instead of errors?"
- "What's 'missing not at random' (MNAR), and why does it matter which rows you drop during cleaning? Give an example where dropping rows on two different null-checks changes your conclusions." (This phase's actual incident — the AISent/AISelect double-filter skewing 2024 adoption from 61.8% to 81.8% — is a strong concrete example to have ready.)

**SQL:**
- "Why does `100 * SUM(x) / COUNT(*)` sometimes return `0` in SQL when you expect a percentage? How do you fix it?"
- "What's the difference between `COUNT(*) FILTER (WHERE ...)` and `SUM(CASE WHEN ... THEN 1 ELSE 0 END)`? Which do you prefer and why?"
- "What's `CREATE TABLE ... AS SELECT` (CTAS) used for, and when would you choose it over transforming data in application code?"
- "How do you decide which columns should be `NOT NULL` when designing a table derived from another table?"

**Data engineering / pipeline design:**
- "You're building a table that an LLM will generate SQL against. What makes a schema 'good' for that use case versus a schema designed for normal application use?"
- "How do you validate that a data transformation/refresh pipeline produced correct output, without a full test suite? What did you actually check?" (Answer: reproduced a known ground-truth aggregate — the 61.8% -> 78.5% figure from the original analysis — rather than just checking row counts.)
- "Describe a time cleaning logic that looked correct actually introduced bias. What was the mechanism, and how did you catch it?"

**Practical/environment:**
- "How does Python virtual environment activation work — is it tied to a directory or a shell session? What problems can that cause in a multi-project workspace?"
