# Akka Sample: RAG Sample

A single retrieval-augmented generation agent accepts a natural-language question, fetches the most relevant passages from an in-process vector store, and returns a grounded answer with citations. Every answer carries a groundedness score so the caller can see immediately whether the response is anchored in retrieved evidence or not.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: an on-decision evaluator that scores every answer for retrieval groundedness — confirming that each claim in the answer maps to a retrieved passage rather than to the model's parametric memory.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The vector store runs in-process using cosine similarity over a seeded document corpus loaded from `src/main/resources/corpus/`.

## Generate the system

```sh
cp -r ./single-agent.general.rag-baseline  ~/my-projects/rag-baseline
cd ~/my-projects/rag-baseline
```

(Optional) Edit `SPEC.md` to point at a different seeded corpus (e.g., swap the product-manual passages for API reference docs or internal runbooks).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **AnswerAgent** — an AutonomousAgent that receives a question plus a set of retrieved passages as task attachments and returns a typed `GroundedAnswer`.
- **QueryWorkflow** — orchestrates retrieve → answer → eval per submitted question.
- **QueryEntity** — an EventSourcedEntity holding the per-query lifecycle.
- **PassageRetriever** — a Consumer that subscribes to `QuerySubmitted` events, calls the in-process vector store, and emits `PassagesRetrieved` back to the entity.
- **QueryView + QueryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded corpus by replacing the JSONL file under `src/main/resources/corpus/passages.jsonl` after generation; the in-process vector store indexes it at startup.
- `SPEC.md §5` — extend `GroundedAnswer` with domain-specific fields (e.g., `confidenceLevel`, `sourceCollection`, `followUpQuestions`).
- `prompts/answer-agent.md` — narrow the agent's role to a specific domain (a technical deployer would constrain it to API reference material; a support deployer to known-issue articles).
- `eval-matrix.yaml` — increase the groundedness threshold or add a second eval dimension such as answer completeness.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a question → passages are retrieved → the agent answers → a groundedness score appears on the same UI card.
2. A question that retrieves zero passages returns a well-formed `GroundedAnswer` with `citations` empty and a groundedness score of 1 — the agent does not hallucinate coverage.
3. An answer where every cited passage id matches a retrieved passage scores 4–5; an answer with fabricated passage ids scores 1–2.
4. The live SSE stream delivers one event per state transition; a browser that opens the stream mid-session sees the current state immediately on the next event.

## License

Apache 2.0.
