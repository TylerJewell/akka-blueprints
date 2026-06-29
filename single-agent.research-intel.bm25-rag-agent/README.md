# Akka Sample: RAG (BM25)

A single retrieval-augmented agent answers natural-language questions about technical documentation by first running a BM25 lexical search over an indexed corpus, then passing the retrieved passages as task attachments to the answering agent. The query and retrieved passages travel as structured inputs — never as ad-hoc string interpolation — so the grounding chain is auditable.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: a `before-agent-response` guardrail that verifies each answer cites only passages that were actually retrieved and delivered as attachments in the same task.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The BM25 index is built in-process from a bundled JSONL corpus at startup; no search server is needed.

## Generate the system

```sh
cp -r ./single-agent.research-intel.bm25-rag-agent  ~/my-projects/bm25-rag-agent
cd ~/my-projects/bm25-rag-agent
```

(Optional) Edit `SPEC.md` to point at a different document corpus — replace the bundled transformers-docs JSONL with your own passage file, or extend the `RetrieverTool` to call a real search index.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CorpusQueryAgent** — an AutonomousAgent that receives the user's question and a set of BM25-retrieved passages (each delivered as a task attachment) and returns a typed `QueryAnswer`.
- **QueryWorkflow** — orchestrates index → retrieve → answer → eval per submitted question.
- **QueryEntity** — an EventSourcedEntity holding the per-query lifecycle.
- **BM25Retriever** — a Consumer that subscribes to `QuerySubmitted` events, runs BM25 over the in-process index, and writes the retrieved passages back to the entity via `attachPassages`.
- **CitationGuardrail** — the before-agent-response hook. Verifies every cited passage id in the answer was actually delivered as an attachment in the same task.
- **QueryView + QueryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded corpus for your own JSONL file (field schema: `{ "passageId", "title", "body", "url" }`).
- `SPEC.md §5` — extend `QueryAnswer` with domain-specific fields (e.g., `confidenceLabel`, `topicTag`, `sourceVersion`).
- `prompts/corpus-query-agent.md` — constrain the agent to a narrower scope (for example, restrict it to Python API documentation only and reject questions about unrelated topics).
- `eval-matrix.yaml` — replace the in-process BM25 scorer with a call to an external ranking endpoint by naming it in the implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a question → passages are retrieved → the agent answers → the answer appears in the UI citing real retrieved passage ids.
2. The agent returns an answer citing a passage id that was not in the retrieved set — the `before-agent-response` guardrail rejects it — the agent retries — a well-grounded answer lands.
3. A query whose retrieved passages contain no relevant content receives an answer with `answerType = NO_ANSWER` and a rationale; no hallucinated content appears.
4. The grounding score chip on every answer card shows 1–5; cards with score ≤ 2 are flagged for review.

## License

Apache 2.0.
