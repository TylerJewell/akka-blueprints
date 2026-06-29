# Akka Sample: Memory Bank

A single memory-agent stores, searches, and retrieves contextual facts on behalf of a user or calling system. The agent accepts `remember` and `recall` requests, persists structured memory entries in an event-sourced store, and returns ranked results for recall queries — all with a PII sanitizer running between the raw input and any LLM call.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: a PII sanitizer that strips identifying information from memory content before the agent ever indexes or reasons over it.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the memory store is in-process and the agent's tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.general.memory-bank  ~/my-projects/memory-bank
cd ~/my-projects/memory-bank
```

(Optional) Edit `SPEC.md` to adjust the retention policy, add a custom memory namespace, or swap the recall ranking strategy.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **MemoryAgent** — an AutonomousAgent that interprets `remember` and `recall` requests, stores structured `MemoryEntry` records, and returns a ranked `RecallResult` list.
- **MemoryWorkflow** — orchestrates sanitize-wait → store/recall per memory operation.
- **MemoryEntity** — an EventSourcedEntity holding the per-namespace memory lifecycle.
- **MemorySanitizer** — a Consumer that subscribes to `MemoryEntrySubmitted` events, redacts PII from the raw content, and emits `MemoryEntrySanitized` back to the entity.
- **MemoryView + MemoryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — adjust the seeded memory namespaces (e.g., swap `personal-assistant` for `customer-support` or `coding-assistant`).
- `SPEC.md §5` — extend `MemoryEntry` with domain-specific fields (e.g., `projectId`, `sessionId`, `expiresAt`).
- `prompts/memory-agent.md` — narrow the agent's recall ranking logic (a coding assistant deployer might prioritise recency; a research assistant might prioritise semantic relevance).
- `eval-matrix.yaml` — wire a production PII library (e.g., a presidio-style pipeline) under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user stores a fact → it is sanitized → indexed → recalled accurately in a subsequent query.
2. The agent returns a malformed recall result (mock LLM path) → the system handles the failure gracefully and the caller receives a clear error, not a partial payload.
3. PII strings stored in a memory entry never appear in the LLM call log; only the redacted form does.
4. A user queries with a broad term; the ranked result list returns the most relevant entries first based on the stored content.

## License

Apache 2.0.
