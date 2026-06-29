# Akka Sample: Agentic Telegram AI Bot

A Router plans a Telegram user's request, dispatches each step to one of several tool-executor agents — WebLookup, ContactBook, CalendarQuery, NoteSaver — tracks progress on a session ledger plus a tool ledger, and replies to the user when all steps are resolved. Demonstrates the **planner-executor** coordination pattern with embedded governance over a dynamic, multi-tool chat assistant.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** The Telegram inbound channel, web lookup, contact book, calendar, and note-saver surfaces are all simulated inside the same Akka service using seeded fixtures.

## Generate the system

```sh
cp -r ./planner-executor.general.telegram-tool-bot  ~/my-projects/telegram-tool-bot
cd ~/my-projects/telegram-tool-bot
```

(Optional) Edit `SPEC.md` to change the bot persona, model provider, or any agent's behaviour.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **RouterAgent** — AutonomousAgent that reads the incoming Telegram message, maintains a session ledger (intent, tool plan, current dispatch) and a tool ledger (each tool call's attempt, verdict, returned content), and decides the next step on every loop tick.
- **WebLookupAgent** — AutonomousAgent that answers web-search subtasks from seeded fixtures.
- **ContactBookAgent** — AutonomousAgent that queries a contact fixture store and returns matching contact records.
- **CalendarQueryAgent** — AutonomousAgent that returns availability and event data from fixture calendars.
- **NoteSaverAgent** — AutonomousAgent that persists short note payloads to a fixture notes store.
- **SessionWorkflow** — Workflow with a plan → dispatch-guarded → execute → sanitize → record → decide loop, replan branch, and terminal exit states.
- **SessionEntity** — EventSourcedEntity holding the session lifecycle, both ledgers, and the final reply.
- **BotControlEntity** — EventSourcedEntity holding the operator pause flag. Single instance keyed by `"global"`.
- **MessageQueue** — EventSourcedEntity acting as an audit log for inbound messages.
- **SessionView** — projection used by the UI.
- **MessageConsumer** — Consumer subscribing to MessageQueue events and starting a SessionWorkflow per message.
- **MessageSimulator** — TimedAction dripping a sample Telegram message every 75 seconds.
- **StaleSessionMonitor** — TimedAction marking sessions stuck in EXECUTING past 4 minutes as STALE.
- **SessionEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the message prompts the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Session` record fields (e.g., add `confidenceScore`).
- `prompts/router.md` — narrow the router to a specific domain (e.g., customer-support-only).
- `eval-matrix.yaml` — add an `eval-event` control if you want per-tool-call quality scoring.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Send a message requesting a web lookup → router plans, dispatches to WebLookup, completes with a reply.
2. Inject a tool-policy violation → guardrail blocks the offending dispatch; router records the block and replans.
3. Trigger the operator pause → no new dispatches start; in-flight tool calls finish; session moves to `PAUSED`.
4. A tool result containing a secret-shaped string is scrubbed before it reaches the router's next prompt.

## License

Apache 2.0.
