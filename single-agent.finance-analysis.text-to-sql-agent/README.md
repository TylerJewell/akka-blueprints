# Akka Sample: Text-to-SQL Agent

A single SQL-generation agent accepts a natural-language question, translates it into a SQL query, and executes it against a SQLite receipts database. The answer — a structured result set plus a prose summary — is returned to the caller. The query never touches the database until it has passed a before-tool-call guardrail that rejects injection patterns and destructive statements. Customer names that appear in the result set are redacted before they reach the response.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a before-tool-call guardrail that validates the generated SQL before execution, and a PII sanitizer that redacts customer identifiers from the result set before the response is returned to the user.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The SQLite receipts database is embedded in the running JVM; no separate database process is required.

## Generate the system

```sh
cp -r ./single-agent.finance-analysis.text-to-sql-agent  ~/my-projects/text-to-sql-agent
cd ~/my-projects/text-to-sql-agent
```

(Optional) Edit `SPEC.md` to point at a different SQLite schema (for example, swap the receipts table for an orders or invoices schema, or add a `line_items` join table).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **SqlGeneratorAgent** — an AutonomousAgent that receives a natural-language question plus the receipts-table schema as a task attachment and returns a typed `QueryResult` carrying the generated SQL and the query answer rows.
- **QueryWorkflow** — orchestrates validate → execute → sanitize per submitted question.
- **QueryEntity** — an EventSourcedEntity holding the per-query lifecycle.
- **SqlGuardrail** — a before-tool-call hook that validates generated SQL before the agent's `execute_sql` tool fires: rejects `DROP`, `DELETE`, `UPDATE`, `INSERT`, `TRUNCATE`, and any statement containing a classic injection signature.
- **ResultSanitizer** — a Consumer that subscribes to `QueryExecuted` events, redacts customer names and emails from result rows, and emits `ResultSanitized` back to the entity.
- **QueryView + QueryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded questions for your own domain (for example, point at an invoices schema or a payroll table).
- `SPEC.md §5` — extend `QueryResult` with domain-specific fields (for example, add `currency`, `fiscalPeriod`, or `costCenter`).
- `prompts/sql-generator.md` — narrow the agent's role to a specific schema; add table-join rules or business-unit filters for your deployment.
- `eval-matrix.yaml` — wire a more sophisticated SQL parser for the guardrail (for example, a JSqlParser-based AST check) by naming it under the guardrail mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a natural-language question → the agent generates SQL → the guardrail passes it → the database executes it → the result rows appear in the UI with customer names redacted.
2. The agent generates a destructive SQL statement (`DROP TABLE` or `DELETE FROM`) → the before-tool-call guardrail rejects it → the agent retries with a safe `SELECT` → the corrected query executes successfully.
3. A query whose result rows contain email addresses has those addresses replaced with `[REDACTED-EMAIL]` before the response reaches the UI.
4. A question that cannot be answered from the receipts schema is returned with an empty result set and an explanation rather than a hallucinated answer.

## License

Apache 2.0.
