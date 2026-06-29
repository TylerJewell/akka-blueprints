# Akka Sample: Mem0 ReAct Agent

A single ReAct agent answers user questions by reasoning over tools and persisting what it learns across sessions. Before any fact enters long-term memory it passes through a PII sanitizer; a human-on-the-loop monitor surfaces session-level memory drift for deployer review.

Demonstrates the **single-agent** coordination pattern with two governance mechanisms: a PII sanitizer that cleans personal facts before they are written to the memory store, and a human-on-the-loop runtime monitor that tracks cumulative memory growth and signals drift to a deployer dashboard.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The memory store runs in-process as an in-memory key-value structure; no external Mem0 service, no vector database.

## Generate the system

```sh
cp -r ./single-agent.general.react-with-longterm-memory  ~/my-projects/mem0-react-agent
cd ~/my-projects/mem0-react-agent
```

(Optional) Edit `SPEC.md` to add domain-specific tools for the agent — for example, a weather lookup or a product-catalog search — under the component table in Section 4.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **Mem0ReactAgent** — an AutonomousAgent that loops over a tool set (web-search stub, calculator, memory-lookup) until it can answer the user's question. Its system prompt instructs it to persist newly learned user facts via the `store-memory` tool and to recall prior facts via `recall-memories` before reasoning.
- **MemoryWriteWorkflow** — orchestrates sanitize-wait → persist per fact the agent requests to store.
- **MemoryEntity** — an EventSourcedEntity holding one record per stored fact, with full event history for audit.
- **FactSanitizer** — a Consumer that subscribes to `FactStoreRequested` events, redacts PII from the raw fact text, and emits `FactSanitized` back to the entity.
- **SessionEntity** — an EventSourcedEntity holding the conversation turn history and current agent status per session.
- **DriftMonitor** — a Consumer that subscribes to `FactPersisted` events and emits `MemoryDriftSignaled` when cumulative per-user fact count crosses a configurable threshold.
- **MemoryView + SessionView + AgentEndpoint + AppEndpoint** — read models, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — extend the tool list with real integrations (replace the web-search stub with a real HTTP call; add a CRM lookup tool or a calendar tool).
- `SPEC.md §5` — add user-profile fields to `MemoryFact` if your deployer tracks structured attributes beyond free-text facts.
- `prompts/mem0-react-agent.md` — constrain the agent's persona and the domains it is allowed to recall (a healthcare deployer would scope recall to clinical preferences only).
- `eval-matrix.yaml` — replace the regex-based sanitizer with a presidio-style library by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user sends a message containing a personal preference → the agent stores a fact → the fact is sanitized → on the next session the agent recalls it and uses it in its answer.
2. A fact containing a raw email address is stored → the PII sanitizer redacts it → the stored fact contains `[REDACTED-EMAIL]`, never the raw string.
3. When a user's fact count crosses the drift threshold, a `MemoryDriftSignaled` event appears in the monitor view within 1 s of the persisting event.
4. The agent answers a multi-step arithmetic question by invoking the calculator tool, not by guessing.

## License

Apache 2.0.
