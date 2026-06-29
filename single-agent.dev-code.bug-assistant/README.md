# Akka Sample: Bug Assistant

A single bug-resolution agent accepts an incoming bug report, searches a ticketing system and web sources for relevant context, proposes a resolution, and writes the result back to the ticket. A `before-tool-call` guardrail intercepts every ticket-write attempt and validates it before the tool executes, preventing the agent from writing malformed or incomplete updates.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: a `before-tool-call` guardrail that blocks every ticket mutation until it passes a structural and content validity check.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the ticketing store and web-search tool are simulated in-process.

## Generate the system

```sh
cp -r ./single-agent.dev-code.bug-assistant  ~/my-projects/bug-assistant
cd ~/my-projects/bug-assistant
```

(Optional) Edit `SPEC.md` to change the seeded bug catalogue (e.g., swap the Java NullPointerException seeds for Python async-await seeds or Go channel-deadlock seeds).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **BugResolutionAgent** — an AutonomousAgent that receives a bug report as a task, searches for relevant context via simulated tool calls, proposes a resolution, and writes back to the ticket.
- **BugWorkflow** — orchestrates intake → enrich → resolve → close per submitted bug.
- **BugEntity** — an EventSourcedEntity holding the per-bug lifecycle.
- **TicketSyncConsumer** — a Consumer that subscribes to `BugSubmitted` events, fetches initial ticket metadata from the simulated ticketing store, and emits `TicketEnriched` back to the entity.
- **BugView + BugEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- **TicketWriteGuardrail** — validates every ticket-write tool call before it executes.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded bug examples for your own (the JSONL file under `src/main/resources/sample-events/seed-bugs.jsonl` after generation).
- `SPEC.md §5` — extend `Resolution` with project-specific fields (e.g., `affectedVersion`, `targetRelease`, `rootCauseCategory`).
- `prompts/bug-resolution-agent.md` — narrow the agent's role to a specific codebase, language, or team convention.
- `eval-matrix.yaml` — wire a real ticketing client (Jira, Linear, GitHub Issues) by naming it under the guardrail mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a bug report → the ticket is enriched → the agent proposes a resolution → the ticket is updated in `RESOLVED` state with a well-formed resolution body.
2. The agent attempts a ticket write with a missing `resolutionBody` field — the guardrail blocks it — the agent retries — a valid write lands.
3. A bug where all three web-search results return empty content still produces a resolution (agent falls back to general knowledge) with a confidence flag set to `LOW`.
4. The SSE stream delivers one event per state transition; a late-joining browser client receives the full current row on the first event.

## License

Apache 2.0.
