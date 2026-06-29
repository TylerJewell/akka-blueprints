# Akka Sample: Chat With PDF Docs Using AI (Quoting Sources)

A single RAG agent accepts a PDF document and a user question, retrieves the most relevant passages via vector similarity, and returns an answer with inline citations pointing to the exact source passages. The PDF is indexed once; all questions run against the stored index.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: a `before-agent-response` guardrail that confirms every answer cites at least one passage by passage id before the response leaves the agent loop. An unanswerable question is answered with "I cannot find this in the document" rather than a hallucinated citation.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the vector index lives in-process and no external embedding service is required.

## Generate the system

```sh
cp -r ./single-agent.research-intel.rag-pdf-chat  ~/my-projects/rag-pdf-chat
cd ~/my-projects/rag-pdf-chat
```

(Optional) Edit `SPEC.md` to point at a different seeded PDF, raise the passage chunk size, or change the maximum number of cited passages per answer.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PdfChatAgent** — an AutonomousAgent that receives a user question and a set of retrieved passages as a task attachment, and returns a typed `CitedAnswer`.
- **ChatSessionWorkflow** — orchestrates retrieve → answer per user question within a session.
- **PdfDocumentEntity** — an EventSourcedEntity holding the indexed document (passages, embeddings) and the per-document lifecycle.
- **PassageRetriever** — a Consumer that subscribes to `DocumentIndexed` events and pre-warms the in-process index; at query time the workflow calls it directly to fetch ranked passages.
- **ChatSessionView + ChatEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded PDF for your own by pointing `src/main/resources/sample-pdfs/` at a different file and updating the seed metadata.
- `SPEC.md §5` — extend `CitedAnswer` with domain-specific fields (e.g., `confidenceLevel`, `topicTag`).
- `prompts/pdf-chat-agent.md` — constrain the agent's citation behaviour (a legal deployer might require a page-and-paragraph reference; a technical deployer might require a section heading).
- `eval-matrix.yaml` — replace the in-process cosine similarity retriever with an external vector database by naming it under the mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user uploads a PDF → it is chunked and indexed → a question returns an answer with one or more cited passage ids.
2. The agent returns an answer with a fabricated passage id on first try → the `before-agent-response` guardrail rejects it → the agent retries → a valid-citation answer lands.
3. A question that cannot be answered by the document returns an explicit "not found" response rather than a hallucinated passage.
4. Multiple questions against the same PDF all resolve from the same indexed document without re-uploading.

## License

Apache 2.0.
