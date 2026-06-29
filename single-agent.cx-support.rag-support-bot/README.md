# Akka Sample: SupportFlow Lite

A single-agent system that answers customer support queries by retrieving relevant passages from a knowledge base and composing a grounded reply. Customer messages are sanitized before the agent sees them; every candidate reply passes through a grounding check before it leaves the agent loop.

Demonstrates the **single-agent** coordination pattern in the CX-support domain. One `SupportAgent` (AutonomousAgent) handles all reply generation; the surrounding components handle ingestion, retrieval, PII stripping, and reply validation.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The knowledge base lives in-process; there is no external vector store dependency for the baseline.

## Generate the system

```sh
cp -r ./single-agent.cx-support.rag-support-bot  ~/my-projects/rag-support-bot
cd ~/my-projects/rag-support-bot
```

(Optional) Edit `SPEC.md` to load a different knowledge base (e.g., swap the seeded FAQ articles for your product's documentation).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **SupportAgent** — an AutonomousAgent that receives a sanitized customer query plus retrieved knowledge passages as a task attachment and returns a typed `SupportReply`.
- **ConversationWorkflow** — orchestrates sanitize-wait → retrieve → reply → grounding-eval per customer query.
- **ConversationEntity** — an EventSourcedEntity holding the per-conversation lifecycle.
- **MessageSanitizer** — a Consumer that subscribes to `QueryReceived` events, strips PII from the customer message, and emits `QuerySanitized` back to the entity.
- **KnowledgeBase** — an EventSourcedEntity (one per article) storing indexed FAQ articles; passages are retrieved in-process by the workflow's `retrieveStep`.
- **ConversationView + ConversationEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — replace the seeded FAQ knowledge base with your own articles (the JSONL file under `src/main/resources/sample-events/kb-articles.jsonl` after generation).
- `SPEC.md §5` — extend `SupportReply` with product-specific fields (e.g., `ticketId`, `escalationFlag`, `suggestedArticleIds`).
- `prompts/support-agent.md` — narrow the agent's persona and constrain the topics it will address.
- `eval-matrix.yaml` — replace the in-process grounding check with a call to a dedicated hallucination-detection API by naming it in the guardrail mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A customer submits a query → the message is sanitized → relevant passages are retrieved → the agent produces a grounded reply that appears in the UI.
2. The agent returns a reply that references zero knowledge-base passages → the `before-agent-response` guardrail rejects it → the agent retries → a passage-anchored reply lands.
3. Every reply carries a grounding score visible on the conversation card.
4. PII strings in the customer query never appear in the LLM call log; only the redacted form does.

## License

Apache 2.0.
