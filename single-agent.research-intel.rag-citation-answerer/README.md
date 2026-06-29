# Akka Sample: RAG Citation Answerer

A single question-answering agent reads uploaded documents from an in-process vector store and returns a grounded answer with source citations. Each answer names the exact document chunks that supplied the evidence, so readers can trace every claim back to its source.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a PII sanitizer that strips personally identifiable text from documents before they enter the index, and a `before-agent-response` guardrail that confirms every citation in the answer refers to a chunk that was actually retrieved.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The vector index and embedding pipeline run in-process; no external vector database or embedding service is required out of the box.

## Generate the system

```sh
cp -r ./single-agent.research-intel.rag-citation-answerer  ~/my-projects/rag-citation-answerer
cd ~/my-projects/rag-citation-answerer
```

(Optional) Edit `SPEC.md` to point at a different document corpus — for example, substitute internal policy documents for the seeded research-report examples, or swap the keyword-based in-process retriever for a real Vertex AI RAG Engine call by naming your project credentials in the implementation paragraph of `eval-matrix.yaml`.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CitationAnswererAgent** — an AutonomousAgent that receives a question plus retrieved document chunks (passed as a task attachment) and returns a typed `CitedAnswer`.
- **AnswerWorkflow** — orchestrates ingest-wait → retrieve → answer → validate per submitted question.
- **DocumentEntity** — an EventSourcedEntity tracking each uploaded document through ingest, sanitize, chunk, and index states.
- **QueryEntity** — an EventSourcedEntity holding the per-query lifecycle from submitted through answered and evaluated.
- **DocumentSanitizer** — a Consumer that subscribes to `DocumentUploaded` events, strips PII from the raw text, and emits `DocumentSanitized` back to the entity.
- **ChunkIndexer** — a Consumer that subscribes to `DocumentSanitized` events, splits the text into chunks, computes keyword-based term-frequency vectors in-process, and stores them.
- **CitationGuardrail** — validates that every citation `chunkId` in the agent's answer appears in the set of retrieved chunks passed to that task.
- **QueryView + DocumentView + QueryEndpoint + AppEndpoint** — read models, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded document corpus (three synthetic research reports under `src/main/resources/sample-events/seed-documents.jsonl`) for your own documents.
- `SPEC.md §5` — extend `CitedAnswer` with domain-specific confidence fields or add a `CitationStyle` enum (inline, footnote, reference-list).
- `prompts/citation-answerer.md` — narrow the agent's persona (an enterprise deployer might limit it to answering only from documents uploaded by the authenticated tenant; a public knowledge-base deployer might widen the disclaimer language).
- `eval-matrix.yaml` — replace the in-process keyword retriever with a real embedding-based retriever by adding the credentials reference in the implementation paragraph of the sanitizer entry.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user uploads a document → it is sanitized and indexed → the user asks a question → a cited answer appears in the UI with every citation traceable to the uploaded document.
2. The agent cites a `chunkId` that was not among the retrieved chunks on a first iteration → the `before-agent-response` guardrail rejects it → the agent retries and produces an answer with valid citations.
3. A document containing PII is uploaded → the indexed chunks never contain the raw PII tokens → a question whose answer would require citing the PII returns the redacted form.
4. A question asked against an empty corpus returns a well-formed `CitedAnswer` with `confidence = LOW` and zero citations.

## License

Apache 2.0.
