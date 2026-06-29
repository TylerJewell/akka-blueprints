# Akka Sample: Personal Assistant

A single personal-assistant agent reads the user's calendar, task list, and reminder state, then creates, updates, or completes items in response to a natural-language request. The request rides into the agent as a task instruction; writes to calendar and tasks are routed through a `before-tool-call` guardrail before any stateful mutation occurs.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a PII sanitizer that strips personal identifiers from the agent's context before the model call, and a `before-tool-call` guardrail that validates every proposed calendar or task write before it executes.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — calendar and task data live in-process; there are no external calendar or task-management API calls.

## Generate the system

```sh
cp -r ./single-agent.planning-travel.personal-assistant  ~/my-projects/personal-assistant
cd ~/my-projects/personal-assistant
```

(Optional) Edit `SPEC.md` to swap the seeded calendar and task examples for your own (e.g., replace the travel-planning events with a meeting-management or project-tracking scenario).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **AssistantAgent** — an AutonomousAgent that accepts a natural-language request and the user's current calendar/task snapshot as a task attachment, then returns a structured `AssistantAction` describing what to create, update, or complete.
- **AssistantWorkflow** — orchestrates sanitize-wait → act → confirm per submitted request.
- **AssistantEntity** — an EventSourcedEntity holding the per-request lifecycle.
- **ContextSanitizer** — a Consumer that subscribes to `RequestSubmitted` events, redacts PII from the assistant's context snapshot, and emits `ContextSanitized` back to the entity.
- **AssistantView + AssistantEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded request examples for your own scenario (e.g., focus on flight reminders, hotel check-in tasks, or packing-list management).
- `SPEC.md §5` — extend `AssistantAction` with domain-specific fields (e.g., `travelLeg`, `confirmationNumber`, `priority`).
- `prompts/assistant-agent.md` — narrow the agent's role (a deployer focused on trip planning would constrain it to itinerary-related actions only).
- `eval-matrix.yaml` — wire a real PII redactor by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a natural-language request → the context is sanitized → the agent produces an action → the action appears in the UI.
2. The agent proposes a write operation that violates the guardrail's rules → the write is blocked before execution → the agent is informed and retries → a valid action lands.
3. PII strings in the context snapshot never appear in the LLM call log; only the redacted form does.
4. An action that produces an empty or implausible change set receives a low eval score visible on the same UI card.

## License

Apache 2.0.
