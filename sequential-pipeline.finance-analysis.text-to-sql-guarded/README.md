# Akka Sample: Advanced Text-to-SQL Workflow

A single `SqlAgent` translates a natural-language financial question through three task phases — **PARSE → QUERY → FORMAT** — wired by explicit task dependencies. The user submits a question in plain English; the pipeline translates it to SQL, screens the statement before execution, runs it against an in-process financial database, redacts PII from the result set, and returns a structured `QueryReport`.

Demonstrates the **sequential-pipeline** coordination pattern wired with three governance mechanisms: a `before-tool-call` guardrail that blocks destructive SQL statements before they execute, a PII sanitizer that scrubs personal data from raw query results, and an automatic safety halt that terminates the pipeline immediately on DROP/DELETE/UPDATE patterns detected in the generated SQL.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the financial database is an in-process H2 instance seeded at startup; every tool is implemented inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.finance-analysis.text-to-sql-guarded  ~/my-projects/text-to-sql
cd ~/my-projects/text-to-sql
```

(Optional) Edit `SPEC.md` to point at a different financial schema, a different model provider, or additional schema tables.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **SqlAgent** — one AutonomousAgent declaring three Task constants (`PARSE_QUESTION`, `EXECUTE_QUERY`, `FORMAT_RESULTS`); the workflow runs them in order, feeding each typed output forward as the next task's instruction context.
- **QueryPipelineWorkflow** — runs `parseStep → queryStep → formatStep`. Each step calls `runSingleTask`, writes the typed result back onto `QueryEntity`, then advances. An automatic safety halt in `parseStep` terminates the workflow immediately if the generated SQL contains destructive patterns.
- **QueryEntity** — an EventSourcedEntity holding the per-query lifecycle (`QuestionParsed`, `QueryExecuted`, `ResultsFormatted`, `SafetyHaltFired`).
- **ParseTools / QueryTools / FormatTools** — three function-tool classes registered on the agent, one per phase. The `before-tool-call` guardrail enforces that each tool is only callable in its own phase.
- **SqlGuardrail** — the runtime check that backs the phase-dependency contract. A tool call referencing a phase whose precondition has not yet been recorded on the entity is rejected before the tool runs.
- **SafetyHalt** — inspects the generated SQL for DROP, DELETE, UPDATE, INSERT, ALTER, TRUNCATE, and GRANT patterns. If any are found, the halt fires before the query reaches the database, records `SafetyHaltFired` on the entity, and ends the workflow with a structured reason.
- **PiiSanitizer** — runs inside `formatStep` after the raw result set arrives. Applies regex and lookup-based redaction rules to replace account numbers, SSNs, and names appearing in result rows. The sanitized rows, not the raw rows, are written onto the entity.
- **QueryView + QueryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded question set under `src/main/resources/sample-events/questions.jsonl` to match your financial schema and demo audience.
- `SPEC.md §4` and `prompts/sql-agent.md` — narrow the agent's role (e.g., constrain it to read-only analytical queries, to a single schema owner, or to a specific GL account range) by tightening the system prompt and adjusting the schema description.
- `SPEC.md §5` — extend the typed outputs (`ParsedQuery`, `RawResult`, `QueryReport`) with domain-specific fields such as currency, fiscal period, or cost-centre codes. The phase-gating guardrail does not need editing — it checks recorded-phase preconditions, not field shapes.
- `eval-matrix.yaml` — wire a real PII detection library (replace the regex stub with a named-entity recogniser) by editing the `S1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a read-only financial question → `PARSE` runs → `QUERY` runs → `FORMAT` runs → a typed `QueryReport` lands in the UI within ~60 s. Every transition is visible in real time.
2. The mock LLM generates a SQL statement containing `DELETE FROM transactions`. The safety halt fires before the query step begins; the entity records `SafetyHaltFired`; the UI shows the halt reason; the database is never touched.
3. A query returns rows containing a raw SSN and account number. The PII sanitizer masks both fields; the `QueryReport` written to the entity carries the redacted values; the raw values never appear in the UI or the API response.
4. A phase-order violation (QUERY-phase tool called during PARSE) is caught by `SqlGuardrail`, logged as `GuardrailRejected`, and the agent retries within its iteration budget.

## License

Apache 2.0.
