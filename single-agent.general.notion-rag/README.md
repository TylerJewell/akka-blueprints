# Akka Sample: Notion AI Assistant Generator

A single-agent system that builds a custom Q&A chatbot bound to a specific Notion database. The agent retrieves rows from the database, constructs a grounded answer, and a `before-agent-response` guardrail verifies the answer is anchored in retrieved content before the response reaches the caller.

Demonstrates the **single-agent** coordination pattern in the general domain. One `NotionQueryAgent` (AutonomousAgent) handles every user question; the surrounding components index the Notion schema, run retrieval, and validate that the response is grounded in what was actually fetched.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- A valid Notion integration token with read access to the target database. Supply it the same way as the model-provider key: an existing env var (`NOTION_API_KEY`), an env file, a secrets-store URI, or a one-time prompt. Akka never writes the token value to disk.

## Generate the system

```sh
cp -r ./single-agent.general.notion-rag  ~/my-projects/notion-rag
cd ~/my-projects/notion-rag
```

(Optional) Edit `SPEC.md` to point at a different Notion database ID or swap the seeded schema for your own in Section 3.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **NotionQueryAgent** — an AutonomousAgent that receives a user question plus retrieved Notion rows as a task attachment, and returns a typed `QueryAnswer`.
- **QueryWorkflow** — orchestrates retrieve → answer → validate per submitted question.
- **SessionEntity** — an EventSourcedEntity holding the per-session conversation history and status.
- **NotionRetriever** — a Consumer that subscribes to `QuestionSubmitted` events, queries the Notion database, and emits `RowsRetrieved` back to the entity.
- **GroundingGuardrail** — validates that every claim in the agent's answer cites a row actually returned from Notion.
- **SessionView + QueryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded Notion database schema (the `schema-snapshot.jsonl` file under `src/main/resources/sample-events/` after generation) for your own property names and types.
- `SPEC.md §5` — extend `QueryAnswer` with domain-specific fields (e.g., `confidence`, `sourceRowIds`, `caveat`).
- `prompts/notion-query-agent.md` — narrow the agent's role to a specific database type (a product catalogue deployer would constrain it to listing properties and pricing; a CRM deployer to company and contact fields).
- `eval-matrix.yaml` — wire a real vector-similarity grounding check by naming the embedding library under the guardrail mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a question → Notion rows are retrieved → the agent answers → the grounding guardrail passes → the answer appears in the UI with source citations.
2. The agent's first attempt on a question returns claims not present in the retrieved rows → the guardrail rejects it → the agent retries → a grounded answer lands.
3. A question that returns zero matching rows yields a transparent "no rows found" answer with eval score 1; the UI flags the session.
4. The Notion API token never appears in the LLM call log or in any stored entity field.

## License

Apache 2.0.
