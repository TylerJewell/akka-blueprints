# Akka Sample: Agent with Long-Term Memory

A single conversational agent retains and recalls facts across independent sessions. Between turns the agent writes structured memory entries to persistent state; at the start of each session it retrieves relevant memories and injects them into context. PII is scrubbed from every memory entry before it is stored — so identifiers gathered during one session never propagate into future model calls without explicit, sanitized form.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: a PII sanitizer that runs inside a Consumer between the raw conversation turn and the memory-write operation.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — memory state lives in-process and sessions are managed by the Akka runtime.

## Generate the system

```sh
cp -r ./single-agent.general.long-term-memory-agent  ~/my-projects/long-term-memory-agent
cd ~/my-projects/long-term-memory-agent
```

(Optional) Edit `SPEC.md` to change how many memories are retrieved per session, the memory retention window, or the extraction prompt in `prompts/memory-agent.md`.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **MemoryAgent** — an AutonomousAgent that accepts a conversation turn, recalls relevant past memories, and generates a response. At the end of each task, it emits candidate memory entries for persistence.
- **MemoryExtractionWorkflow** — orchestrates sanitize-wait → store per turn that produces new memories.
- **SessionEntity** — an EventSourcedEntity holding the per-session conversation history and status.
- **MemoryEntity** — an EventSourcedEntity holding the agent's persistent memory store for a given user.
- **MemorySanitizer** — a Consumer that subscribes to `MemoryExtracted` events, scrubs PII from raw memory entries, and writes sanitized entries back.
- **MemoryView + SessionView + ConversationEndpoint + AppEndpoint** — read models, REST/SSE APIs, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — adjust the number of memories recalled per turn (`topK`), the retention-window days, or the recency-vs-relevance weighting strategy.
- `SPEC.md §5` — add fields to `MemoryEntry` (e.g., `topic`, `confidence`, `sourceSessionId`) for richer recall.
- `prompts/memory-agent.md` — narrow the agent's persona (a support-desk deployer might limit memory to ticket history; a developer-tools deployer might limit it to project context).
- `eval-matrix.yaml` — wire a real PII redaction library by naming it in the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user starts a new session and sends a message; the agent responds, extracts memory entries, and stores them after sanitization.
2. The user starts a second session; the agent's recall step surfaces facts from the first session and uses them in its reply.
3. A message containing PII strings produces sanitized memory entries; the raw strings never appear in stored memories or future LLM calls.
4. A session that exceeds its turn budget transitions to `CLOSED`; no further messages are accepted.

## License

Apache 2.0.
