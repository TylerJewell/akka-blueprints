# Akka Sample: Corrective RAG Workflow

A retriever agent fetches document chunks for a query; a relevance evaluator scores each chunk against the query; when the retrieved context is insufficient, the workflow falls back to a web-search agent before generating the final answer. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The retrieval store and the web-search surface are both modeled inside the service; the blueprint runs out of the box.

## Generate the system

```sh
cp -r ./evaluator-optimizer.research-intel.corrective-rag-workflow  ~/my-projects/corrective-rag-workflow
cd ~/my-projects/corrective-rag-workflow
```

(Optional) Edit `SPEC.md` to change the relevance threshold, the document corpus, the maximum web-search results, or the retry ceiling.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **RetrieverAgent** — AutonomousAgent that fetches document chunks from the in-process corpus store, given a query string.
- **RelevanceEvaluatorAgent** — AutonomousAgent that scores each retrieved chunk against the query, returns a `RelevanceVerdict` with a per-chunk score and an overall `SUFFICIENT` or `INSUFFICIENT` judgment.
- **WebSearchAgent** — AutonomousAgent that executes a web-search fallback when retrieval is deemed insufficient, returning a ranked list of `WebResult` entries.
- **AnswerSynthesizerAgent** — AutonomousAgent that generates the final answer given a query and the assembled context (retrieved chunks plus any web results).
- **CorrectiveRagWorkflow** — Workflow that runs the retrieve → evaluate → (fallback?) → synthesize pipeline, halting at a configurable retry ceiling if successive retrievals remain insufficient.
- **QueryEntity** — EventSourcedEntity that holds the query lifecycle, every retrieval attempt, every relevance verdict, and the final answer.
- **CorpusStore** — EventSourcedEntity that holds the in-process document chunks and answers chunk-fetch commands.
- **QueryView** — read-side projection that the UI lists and streams via SSE.
- **QueryConsumer** — Consumer that starts a workflow per inbound query submission.
- **QuerySimulator** — TimedAction that drips a sample query every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-retrieval eval event each cycle (control E1).
- **RagEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample queries the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Query` record fields (e.g., raise `maxRetrievalAttempts`, lower the relevance threshold).
- `prompts/retriever.md` — change how the retriever selects and ranks chunks from the corpus store.
- `prompts/relevance-evaluator.md` — tighten or loosen the scoring rubric.
- `prompts/web-search.md` — add a domain filter or result-count cap.
- `prompts/answer-synthesizer.md` — change citation format or answer length.
- `eval-matrix.yaml` — upgrade the web-search guardrail from non-blocking to blocking if you want failed pre-call checks to abort the fallback entirely.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a query with good corpus coverage → answer generated from retrieval alone, no web-search fallback; the expanded view shows each chunk's relevance score and `SUFFICIENT` verdict.
2. Submit a query with no corpus coverage → retrieval scores `INSUFFICIENT`, the web-search guardrail clears, fallback fires, the answer is generated from web results; the full provenance chain is preserved on the entity.
3. The web-search guardrail blocks the fallback when the query contains a disallowed pattern (control H1); the system returns the best available retrieved answer rather than calling the external tool.
4. Each completed evaluation cycle emits an `EvalRecorded` event that surfaces in the App UI's per-attempt timeline.

## License

Apache 2.0.
