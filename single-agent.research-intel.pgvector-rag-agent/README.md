# Akka Sample: RAG Agent

A single retrieval-augmented generation agent embeds a documentation corpus into a pgvector-backed store and answers natural-language questions by retrieving relevant passages and grounding every reply in cited sources. The answer never goes out without citation — a `before-agent-response` guardrail checks that every claim references a passage from the retrieved context before the response leaves the agent's loop.

Demonstrates the **single-agent** coordination pattern in the research-intel domain. One `CorpusQueryAgent` (AutonomousAgent) carries the retrieval and synthesis; the surrounding components ingest documents, index embeddings, and audit answers.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- **Docker Desktop or Docker Engine + Compose v2, running.** The blueprint starts a pgvector container for the embedding store. All other dependencies are in-process.

## Generate the system

```sh
cp -r ./single-agent.research-intel.pgvector-rag-agent  ~/my-projects/pgvector-rag-agent
cd ~/my-projects/pgvector-rag-agent
```

(Optional) Edit `SPEC.md` to point at a different document corpus — swap out the seeded technical documentation JSONL file for your own collection, or adjust the chunking strategy in the `CorpusIngestionConsumer` description.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CorpusQueryAgent** — an AutonomousAgent that receives a question and a list of retrieved passages (passed as a task attachment), synthesises a grounded answer, and cites the source chunk ids.
- **QueryWorkflow** — orchestrates embed-question → retrieve-chunks → answer → eval per submitted question.
- **QuestionEntity** — an EventSourcedEntity tracking the per-question lifecycle: submitted → retrieving → answering → answered → evaluated.
- **CorpusIngestionConsumer** — a Consumer that subscribes to `DocumentAdded` events on `CorpusEntity`, splits the document into chunks, calls the embedding endpoint, and writes the vectors to pgvector.
- **CitationGuardrail** — a `before-agent-response` hook that asserts every answer sentence that makes a factual claim cites at least one chunk id from the retrieved set.
- **AnswerEvaluationScorer** — a deterministic rule-based scorer that checks answer length, citation density, and that no claim references a chunk id outside the retrieved set.
- **QuestionView + QueryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded corpus for your documentation set (the JSONL file under `src/main/resources/sample-events/corpus-documents.jsonl` after generation).
- `SPEC.md §5` — extend `Answer` with domain fields (e.g., `confidenceLevel`, `dataClassification`, `languageCode`).
- `prompts/corpus-query-agent.md` — narrow the agent's role. A security documentation deployer would restrict it to CVE and patch notes; an API-reference deployer would constrain it to method signatures and examples only.
- `eval-matrix.yaml` — extend `CitationGuardrail` checks or replace the rule-based `AnswerEvaluationScorer` with a second agent-based grading step if your domain needs richer quality signals.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user ingests a document corpus → asks a question → an answer with citations appears in the UI.
2. The agent returns an answer that lacks any citation on the first iteration → the `before-agent-response` guardrail rejects it → the agent retries → a cited answer lands.
3. Every recorded answer carries an evaluation score visible on the same UI card.
4. A question about content not present in the corpus receives an explicit "insufficient context" verdict, not a hallucinated answer.

## License

Apache 2.0.
