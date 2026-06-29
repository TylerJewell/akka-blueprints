# Architecture — telegram-tool-bot

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`SessionEndpoint` is the entry point. A message submission writes a `MessageReceived` event to `MessageQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `SessionWorkflow` per message. The workflow drives the planner-executor loop: it asks `RouterAgent` to parse intent and produce a session ledger, then on each iteration asks it to decide the next step. The decision routes to one of four tool executors — `WebLookupAgent`, `ContactBookAgent`, `CalendarQueryAgent`, `NoteSaverAgent`. Every transition emits an event on `SessionEntity`; `SessionView` projects those events into the read model the UI streams via SSE.

Two TimedActions run alongside: `MessageSimulator` drips sample Telegram messages for demo purposes; `StaleSessionMonitor` ticks every 30 s to mark sessions stuck in `EXECUTING` for over 4 minutes as `STALE`.

`BotControlEntity` is a single-instance entity keyed by the literal string `"global"`. Operators flip its pause flag from the UI; the workflow polls the flag before every dispatch.

## Interaction sequence

The sequence diagram traces the happy path (J1). The loop in the middle is the planner-executor cycle — each iteration runs check-pause → propose → guardrail → dispatch → sanitize → record → decide. The diagram condenses the four tool executors into a single participant for readability; in code the workflow's `dispatchStep` switches on `ToolDispatch.tool` to call the matching agent.

## State machine

`SessionEntity` has six states. `PLANNING` is the initial state. `EXECUTING` is the loop state — most events fire here without changing the status. A session lands in one of four terminal states: `COMPLETED` (happy path; router produced a reply), `FAILED` (router exhausted its replan or failure budget), `PAUSED` (operator pressed Pause), or `STALE` (no progress for 4 minutes).

## Entity model

`SessionEntity` is the system's source of truth; every transition writes one of nine event types. `BotControlEntity` carries the operator pause flag. `MessageQueue` is the audit log of inbound messages. `SessionView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeouts: `planStep` 60 s, `proposeStep` 45 s, `dispatchStep` 120 s, `decideStep` 45 s, `replyStep` 60 s.
- Replan budget: 2 consecutive `Replan` outputs; the third becomes `Fail`.
- Failure budget: 3 consecutive `Continue` outputs on the same `(tool, subtask)`; the fourth becomes `Fail`.
- Idempotency: `(text, chatId)` over a 10 s window deduplicates `POST /api/sessions`.
- Stale detection: `StaleSessionMonitor` every 30 s; sessions `EXECUTING` for > 4 minutes are marked `STALE`.
- Pause poll: synchronous read of `BotControlEntity` at the top of every loop iteration; no caching.
