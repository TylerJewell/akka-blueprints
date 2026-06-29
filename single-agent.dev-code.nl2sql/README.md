# Akka Sample: NL2SQL

A single-agent system that accepts a natural-language question, translates it into a read-only SQL query, executes it against a PostgreSQL database, and returns the result as structured JSON. The agent never writes to, alters, or drops database objects — a before-tool-call guardrail enforces a read-only role on every generated query, and an automatic safety halt terminates any task that attempts a write or DDL statement.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a SQL-safety guardrail that intercepts every tool call before execution and an automatic-safety-halt that stops the task immediately when a write or DDL statement is detected.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- **Docker Desktop or Docker Engine with Compose v2, running.** The blueprint starts a local PostgreSQL container to back the query executor; without a running Docker daemon the service will not start.

## Generate the system

```sh
cp -r ./single-agent.dev-code.nl2sql  ~/my-projects/nl2sql
cd ~/my-projects/nl2sql
```

(Optional) Edit `SPEC.md` to point at your own database schema or to seed the schema registry with your table/column definitions.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **SqlQueryAgent** — an AutonomousAgent that accepts a natural-language question, uses a schema-lookup tool and a query-execution tool, generates a SQL SELECT, runs it, and returns a typed `QueryResult`.
- **QueryWorkflow** — orchestrates schema-fetch → translate-and-run → eval per submitted question.
- **QueryEntity** — an EventSourcedEntity holding the per-query lifecycle.
- **SchemaRegistryConsumer** — a Consumer that subscribes to `QuerySubmitted` events, fetches relevant schema fragments from the registry, and emits `SchemaAttached` back to the entity.
- **QueryView + QueryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- **SqlSafetyGuardrail** — a before-tool-call guardrail that rejects any query containing write or DDL keywords before the database tool fires.
- **QueryResultScorer** — a deterministic rule-based scorer that evaluates every result for row-count reasonableness and column-coverage.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — replace the seeded schema with your own (edit `src/main/resources/sample-events/schema-registry.jsonl` after generation).
- `SPEC.md §5` — extend `QueryResult` with pagination, total-row-count, or a query-plan field.
- `prompts/sql-query-agent.md` — narrow the agent's dialect (PostgreSQL 15 window functions only, for example) or restrict it to a specific set of tables.
- `eval-matrix.yaml` — wire a real read-only Postgres role by naming it in the guardrail's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a natural-language question → the agent translates it → the query runs → the result appears in the UI.
2. The agent generates a query containing `DELETE` on its first iteration → the safety halt fires immediately → the task terminates; no database call is made.
3. A valid query that returns zero rows receives a result-scorer flag (score 2) with a rationale noting the empty result set.
4. The agent generates a structurally valid but overly broad `SELECT *` → the before-tool-call guardrail rejects it → the agent narrows the column list on retry.

## License

Apache 2.0.
