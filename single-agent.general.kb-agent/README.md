# Akka Sample: Agent with Knowledge Base

A single retrieval-augmented agent accepts a natural-language question, searches a pre-loaded Knowledge Base of documents, and returns a grounded answer that cites only the passages it retrieved. Questions that fall outside the Knowledge Base receive a transparent "not found" response rather than a hallucinated one.

Demonstrates the **single-agent** coordination pattern with one governance mechanism: an on-decision evaluator that scores every answer for groundedness — verifying that each sentence in the response traces back to a retrieved passage.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The Knowledge Base is seeded from JSONL files bundled in `src/main/resources/` and loaded on startup. No external vector store, no cloud retrieval service.

## Generate the system

```sh
cp -r ./single-agent.general.kb-agent  ~/my-projects/kb-agent
cd ~/my-projects/kb-agent
```

(Optional) Edit `SPEC.md` to swap the seeded Knowledge Base for your own document corpus (change the JSONL file under `src/main/resources/sample-events/kb-documents.jsonl` after generation) or to tune the retrieval result count.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **KbAnswerAgent** — an AutonomousAgent that receives a question and a set of retrieved passages as a task attachment, then returns a typed `KbAnswer`.
- **QueryWorkflow** — orchestrates retrieve → answer → eval for each question.
- **QueryEntity** — an EventSourcedEntity holding the per-query lifecycle.
- **KbDocumentConsumer** — a Consumer that subscribes to `DocumentIndexed` events and keeps the in-process knowledge index up to date.
- **KbView + QueryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded document corpus; change the number of passages returned per retrieval (`topK`).
- `SPEC.md §5` — add domain-specific fields to `KbAnswer` (e.g., `confidenceHint`, `sourceCollection`, `language`).
- `prompts/kb-answer-agent.md` — narrow the agent's role; for a customer-support deployer, constrain it to product documentation only.
- `eval-matrix.yaml` — adjust the groundedness threshold (minimum citation ratio) to match your deployment's trust requirements.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user asks a question covered by the Knowledge Base → passages are retrieved → an answer citing those passages appears in the UI.
2. A user asks a question with no matching passages → the agent returns a transparent "not found" response; the groundedness evaluator scores it accordingly.
3. Every recorded answer has an on-decision groundedness score visible on the same UI card.
4. An answer whose sentences do not trace back to retrieved passages scores ≤ 2 and is flagged in the UI for review.

## License

Apache 2.0.
