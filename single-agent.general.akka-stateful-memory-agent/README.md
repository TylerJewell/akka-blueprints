# Akka Sample: Hello World Stateful Agent

A single conversational agent that maintains persistent memory blocks — a `persona` block describing itself and a `human` block accumulating facts about the user. Every turn the agent reads both blocks, responds, and then rewrites them based on what it learned. Restart the service; the conversation resumes exactly where it left off.

Demonstrates the **single-agent** coordination pattern with two governance mechanisms: a PII sanitizer that scrubs user messages before the agent writes them into the human memory block, and a periodic fairness-drift evaluator that flags when the human block's accumulated profile has drifted toward demographic stereotyping.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — memory blocks are stored in-process and the agent's tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.general.akka-stateful-memory-agent  ~/my-projects/stateful-agent
cd ~/my-projects/stateful-agent
```

(Optional) Edit `SPEC.md` to rename or reword the initial persona block (e.g., give the agent a different name or role description).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **MemoryAgent** — an AutonomousAgent that reads both memory blocks as context, responds to the user's message, and updates the `persona` and `human` blocks based on what it learned.
- **ConversationEntity** — an EventSourcedEntity holding per-conversation history and the two live memory blocks.
- **MemoryWriterConsumer** — a Consumer that subscribes to `TurnCompleted` events, applies the agent's proposed memory patches, and sanitizes any PII before persisting them to the entity.
- **ConversationView + ConversationEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seed persona text to match a different role (customer support, coding assistant, research aide).
- `SPEC.md §5` — add fields to `MemoryBlock` for domain-specific facts (e.g., `preferredLanguage`, `currentProject`).
- `prompts/memory-agent.md` — adjust the update rules the agent uses to decide what is worth remembering.
- `eval-matrix.yaml` — replace the built-in drift heuristic with a call to a real bias-detection library by naming it in the evaluator's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user sends three messages; after each one the human block shows updated facts; after a service restart the conversation resumes with the same blocks.
2. A user message containing an email address is saved to the human block as `[REDACTED-EMAIL]`; the raw form never appears in the agent's context window.
3. After ten turns where every message describes the user in terms of a single demographic attribute, the drift evaluator flags the session with a `drift-risk` annotation.
4. The persona block never loses its initial content across turns — only new information is merged in.

## License

Apache 2.0.
