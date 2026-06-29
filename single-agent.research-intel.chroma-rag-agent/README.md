# Akka Sample: RAG Using ChromaDB

A single CodeAgent answers natural-language questions against a document corpus indexed in ChromaDB. The agent retrieves the top-k most relevant passages by embedding similarity, constructs an answer grounded in those passages, and returns a `RagAnswer` carrying both the response text and the citations. A `before-agent-response` guardrail verifies that every cited chunk actually appears in the retrieved context before the answer leaves the agent loop.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: a citation-groundedness guardrail that runs on every turn and rejects answers that assert facts not traceable to a retrieved chunk.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. ChromaDB runs embedded (in-process via the Java client in local mode); the docs corpus is seeded from bundled JSONL at startup. No external vector-store process is required.

## Generate the system

```sh
cp -r ./single-agent.research-intel.chroma-rag-agent  ~/my-projects/chroma-rag-agent
cd ~/my-projects/chroma-rag-agent
```

(Optional) Edit `SPEC.md` to point at a different document corpus — replace the bundled `docs-corpus.jsonl` path with your own JSONL or change the embedding model reference in the implementation directives.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **RagAgent** — an AutonomousAgent that accepts a question as a task, retrieves the top-k passages from ChromaDB, and returns a typed `RagAnswer`.
- **QueryWorkflow** — orchestrates index-ready-wait → retrieve → answer → eval per submitted question.
- **QueryEntity** — an EventSourcedEntity holding the per-query lifecycle.
- **CorpusIndexer** — a Consumer that subscribes to corpus-loaded events and drives the ChromaDB embedding pipeline.
- **RagView + QueryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded docs corpus (Akka documentation excerpts) for your own JSONL file.
- `SPEC.md §5` — extend `RagAnswer` with domain-specific fields (e.g., `topicCategory`, `confidenceLabel`).
- `prompts/rag-agent.md` — narrow the agent's role (a support deployer might constrain it to answer only from the indexed product docs, refusing out-of-corpus questions).
- `eval-matrix.yaml` — wire a real embedding service by naming it under the guardrail mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a question → the corpus is retrieved → the agent answers → citations appear in the UI.
2. The agent returns an answer whose citations do not match the retrieved chunks → the `before-agent-response` guardrail rejects it → the agent retries → a grounded answer lands.
3. A question with no relevant passages scores a groundedness eval of 1, flagging the card.
4. Cited chunk IDs in every answer match chunks that are present in the retrieved context list.

## License

Apache 2.0.
