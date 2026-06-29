# Akka Sample: CodeSearch Demo Agent

A single repository Q&A agent indexes a code repository and answers natural-language questions by returning specific file paths and line numbers. Questions arrive as a query request; the agent retrieves relevant code chunks through a semantic vector index, then produces a grounded answer with exact source references.

Demonstrates the **single-agent** coordination pattern in the dev-code domain. One `CodeSearchAgent` (AutonomousAgent) carries the entire retrieval and answer step; the surrounding components ingest the repository, maintain the index, and audit the agent's output. One governance mechanism wraps the agent: a secret sanitizer that redacts hardcoded credentials and tokens from retrieved code chunks before the LLM ever sees them.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The vector index is in-process; the seeded repository corpus ships as JSONL in `src/main/resources/`.

## Generate the system

```sh
cp -r ./single-agent.dev-code.code-search-rag-agent  ~/my-projects/code-search
cd ~/my-projects/code-search
```

(Optional) Edit `SPEC.md` to point at a different code corpus or extend the chunk size for larger files.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CodeSearchAgent** — an AutonomousAgent that receives a natural-language question plus code-chunk attachments and returns a typed `SearchAnswer` with file paths and line numbers.
- **QueryWorkflow** — orchestrates sanitize-chunks → answer per submitted query.
- **QueryEntity** — an EventSourcedEntity holding the per-query lifecycle.
- **ChunkSanitizer** — a Consumer that subscribes to `QuerySubmitted` events, redacts secrets from retrieved chunks into `SanitizedChunks`, and emits `ChunksSanitized` back to the entity.
- **ChunkIndexView + QueryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded code corpus for a real repository directory by pointing `src/main/resources/sample-events/corpus.jsonl` at your chunks.
- `SPEC.md §5` — extend `SearchAnswer` with additional fields (e.g., `confidence`, `language`, `testCoverage`).
- `prompts/code-search-agent.md` — narrow the agent's scope (a security team deployer might instruct it to only answer questions about authentication and authorization code paths).
- `eval-matrix.yaml` — wire a real secret-scanning library (e.g., a truffleHog-style scanner) by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user asks a code question → relevant chunks are retrieved and sanitized → the agent returns an answer with file paths and line numbers → the answer appears in the UI.
2. A chunk containing a real API key string is retrieved; the key never appears in the LLM call log; only the redacted form does.
3. The agent returns an answer referencing a file path that was not in the retrieved chunks — the answer is flagged with a low grounding score visible on the UI card.
4. A query whose retrieved chunks contain no relevant code produces a graceful "not found" answer with a grounding score of 1.

## License

Apache 2.0.
