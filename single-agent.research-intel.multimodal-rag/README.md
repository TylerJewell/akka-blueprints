# Akka Sample: Multiformat Hybrid RAG

A single research agent accepts a natural-language question and retrieves supporting passages from a corpus that mixes plain text, markdown, extracted PDF pages, and image-derived captions. The agent selects the most relevant chunks, synthesises an answer, and returns a structured `ResearchAnswer` — every claim backed by a cited passage. A `before-agent-response` guardrail enforces citation coverage: any answer that does not anchor every factual claim to a retrieved chunk is rejected before it leaves the agent loop.

Demonstrates the **single-agent** coordination pattern wired to one governance mechanism: a citation guardrail that runs immediately before the agent's response is accepted, ensuring the model cannot assert facts without a passage reference.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The corpus lives in-process as a seeded JSONL file; no external vector database, search index, or embedding service is required to run the blueprint.

## Generate the system

```sh
cp -r ./single-agent.research-intel.multimodal-rag  ~/my-projects/multimodal-rag
cd ~/my-projects/multimodal-rag
```

(Optional) Edit `SPEC.md` to point at a different seeded corpus (e.g., switch the JSONL from a synthetic climate-science excerpt set to a product-documentation set).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ResearchAgent** — an AutonomousAgent that receives a question plus retrieved chunks as task attachments and returns a typed `ResearchAnswer`.
- **QueryWorkflow** — orchestrates index-lookup → rank → answer → citation-check per submitted question.
- **QueryEntity** — an EventSourcedEntity holding the per-query lifecycle.
- **ChunkIndexer** — a Consumer that subscribes to `CorpusChunkAdded` events, extracts text from each format (text passthrough, markdown strip, PDF page text, image caption), and writes a `ChunkRecord` ready for retrieval.
- **QueryView + QueryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded corpus JSONL for your own (the file under `src/main/resources/sample-events/corpus-chunks.jsonl` after generation).
- `SPEC.md §5` — extend `ResearchAnswer` with domain-specific fields (e.g., `confidenceLevel`, `topicCategory`, `sourceDataset`).
- `prompts/research-agent.md` — narrow the agent's role (a biomedical deployer would restrict citation format to PubMed-style; a policy deployer might require statutory section references).
- `eval-matrix.yaml` — wire a stricter citation parser that validates passage hashes or source URLs rather than substring matching.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a question → the system retrieves chunks → the agent returns an answer → every cited chunk id appears in the retrieved set, and the answer is visible in the UI.
2. The agent returns a citation-free factual claim on its first iteration → the `before-agent-response` guardrail rejects it → the agent retries → the revised answer includes proper citations → the UI shows the final answer.
3. A question that matches only image-caption chunks returns a correctly attributed answer with `sourceFormat: image-caption` on those citations.
4. A question submitted with a topic outside the seeded corpus returns an explicit `NO_RESULT` decision with a one-sentence rationale rather than a hallucinated answer.

## License

Apache 2.0.
