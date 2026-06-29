# Akka Sample: Adaptive/Corrective RAG

A retriever agent fetches documents for a user query; a relevance grader scores each document and discards low-scoring ones; if no document clears the threshold the pipeline rewrites the query or falls back to web search; a generator agent produces the final answer; a hallucination grader checks the answer against the retained documents and either accepts it or routes back for re-generation.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; the document corpus and web-search fallback are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.research-intel.self-correcting-rag  ~/my-projects/self-correcting-rag
cd ~/my-projects/self-correcting-rag
```

(Optional) Edit `SPEC.md` to change the document corpus, the relevance threshold, the hallucination tolerance, or the retry cap.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **RetrieverAgent** — AutonomousAgent that fetches candidate documents for a query from the in-memory corpus.
- **RelevanceGraderAgent** — AutonomousAgent that scores each retrieved document for relevance to the query; returns `RELEVANT` or `IRRELEVANT` per document.
- **QueryRewriterAgent** — AutonomousAgent that rewrites a query whose documents all scored `IRRELEVANT`, producing a refined query for a second retrieval pass or a web-search fallback call.
- **GeneratorAgent** — AutonomousAgent that produces a grounded answer from the retained documents.
- **HallucinationGraderAgent** — AutonomousAgent that checks whether the generated answer is supported by the retained documents; returns `GROUNDED` or `HALLUCINATED`.
- **RagWorkflow** — Workflow that orchestrates the retrieve → grade → (rewrite|continue) → generate → hallucination-check loop with configurable retry ceilings.
- **QueryEntity** — EventSourcedEntity that holds the query lifecycle, every retrieval pass, every grading result, and the final answer.
- **CorpusEntity** — EventSourcedEntity that stores the in-memory document corpus and answers document-fetch commands.
- **QueryView** — read-side projection that the UI lists and streams via SSE.
- **QueryConsumer** — Consumer that starts a workflow per submitted query.
- **QuerySimulator** — TimedAction that drips a sample query every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records per-step grading eval events each cycle (control E1).
- **RagEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample queries the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust relevance threshold, hallucination tolerance, or per-query retry ceilings.
- `prompts/relevance-grader.md` — tighten or widen the relevance rubric.
- `prompts/hallucination-grader.md` — change the grounding criteria.
- `eval-matrix.yaml` — tighten enforcement on the hallucination guardrail (currently non-blocking advisory).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a query → pipeline progresses `RETRIEVING` → `GRADING` → `GENERATING` → `ANSWERED` within the retry ceiling.
2. Force all documents to grade `IRRELEVANT` → pipeline invokes `QueryRewriterAgent`, performs a second retrieval pass, and continues; if still no relevant documents after the rewrite ceiling, the query lands in `FAILED_NO_RELEVANT_DOCS`.
3. Force the generator to produce a hallucinated answer → the hallucination grader returns `HALLUCINATED`, the pipeline routes back for re-generation; if the ceiling is hit, the query lands in `FAILED_HALLUCINATION`.
4. Each grading step emits an `EvalRecorded` event that surfaces in the App UI's per-step timeline.

## License

Apache 2.0.
