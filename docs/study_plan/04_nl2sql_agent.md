# Study: NL2SQL Agent with the OpenAI Agents SDK

## Concept

This is the phase that's genuinely new (vs. the pgvector/semantic-retrieval RAG design considered earlier for this same dataset). The pattern here is **text-to-SQL agentic tool use**, not embeddings:

1. The agent receives the user's question plus a system prompt describing the `ai_trends_refined` schema (column names, types, and the value glossary from Phase 1).
2. The agent decides to call a tool — something like `run_sql_query(query: str)` — with a SQL string it generates itself.
3. Your tool code executes that SQL against Postgres (read-only) and returns the rows.
4. The agent reads the results and composes a natural-language answer, ideally citing which numbers it used.

Core OpenAI Agents SDK concepts you'll use, all of which you already touched in the earlier chatbot planning (the `@tool` decorator, `Agent`, `Runner.run`) — the new part is that the tool executes **arbitrary agent-generated SQL** instead of a fixed retrieval query, which raises a real safety question: **never let the agent's SQL run unchecked**. At minimum:
- Reject anything that isn't a `SELECT` (a simple string check, or better, use `sqlglot`/`sqlparse` to verify it parses as a read-only statement).
- Run it against a Postgres role that only has `SELECT` grants on `ai_trends_refined` — not against the `postgres` superuser.
- Cap rows returned (`LIMIT 50`) so a broad/wrong query doesn't flood the model's context.

## Study material

- **Video series (foundation, if OpenAI Agents SDK basics need a refresher):** [OpenAI Agents SDK Tutorial (FULL SERIES)](https://www.youtube.com/watch?v=gFcAfU3V1Zo)
- **Video (Agents SDK basics, shorter):** [Agents SDK from OpenAI! | Full Tutorial](https://www.youtube.com/watch?v=35nxORG1mtg)
- **Written, OpenAI-Agents-SDK-specific text-to-SQL walkthrough (closest match to this phase's exact task):** [Text-to-SQL Agent with the OpenAI Agents SDK and Daytona](https://www.daytona.io/docs/en/guides/openai-agents/text-to-sql-agent-openai-agents-sdk/)
- **Video (concept explainer, uses LangChain instead of OpenAI Agents SDK but the NL2SQL *pattern* — schema-in-prompt, generate SQL, execute, answer — is the same one you're implementing):** [Natural Language to SQL using AI Agent for QA](https://www.youtube.com/watch?v=TF6X1xrPuG8)
- **Video (another LangChain-based walkthrough, useful for seeing the pattern explained a second way):** [Natural Language to SQL | LangChain, SQL Database & OpenAI LLMs](https://www.youtube.com/watch?v=w-eTS8YlbZ4)

Note: several of the NL2SQL videos above use LangChain's SQL agent instead of the OpenAI Agents SDK — that's fine for understanding the *concept* (schema-aware prompting, generate → execute → answer, guarding against unsafe SQL), but this project's actual tool-wiring code should follow the Daytona guide and the earlier chatbot-planning code shape (`@tool` decorator + `Agent` + `Runner.run`), not swap in LangChain.

## Notes (fill in as you go)

_Your system prompt / schema description text, the safety check you implemented, example questions that worked vs. ones the agent got wrong and why._
