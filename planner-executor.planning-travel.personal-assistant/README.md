# Akka Sample: LINE Personal Assistant

A conversational LINE messaging agent that schedules Google Calendar events and sends Gmail messages via natural-language commands. The user sends a LINE message; a planner agent interprets intent, decides which integration to call, and executes it — with a guardrail checking every outbound write before it runs and an optional confirm-before-send gate for email.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- **A valid Google OAuth credential** (client ID + refresh token) with `https://www.googleapis.com/auth/calendar` and `https://mail.google.com/` scopes, and a **LINE Messaging API channel access token**. Supply these via the same key-sourcing options as the LLM key — env vars, env file, secrets-store URI, or one-time session prompt. Akka records only the reference (env-var name, file path, or URI); the values are never written to disk.

## Generate the system

```sh
cp -r ./planner-executor.planning-travel.personal-assistant  ~/my-projects/line-personal-assistant
cd ~/my-projects/line-personal-assistant
```

(Optional) Edit `SPEC.md` to change the confirmation policy, the calendar time zone, or the email signature template.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PlannerAgent** — AutonomousAgent that reads the user's LINE message and produces an `ActionPlan { intent, action, parameters }`. Decides whether to call the Calendar executor, the Gmail executor, or to reply with a clarification request.
- **CalendarExecutorAgent** — AutonomousAgent that turns a `CalendarActionParams` record into a Google Calendar API call (create or list events).
- **GmailExecutorAgent** — AutonomousAgent that turns a `GmailActionParams` record into a Gmail send-message call, subject to the confirmation gate.
- **ConversationWorkflow** — Workflow that drives the plan → guard → (confirm?) → execute → reply loop. Handles the confirm-before-send gate for outbound email.
- **ConversationEntity** — EventSourcedEntity holding each LINE conversation's state, plan, and execution result.
- **ConfirmationEntity** — EventSourcedEntity holding pending outbound email confirmations awaiting user approval via LINE reply.
- **MessageQueue** — EventSourcedEntity that records every inbound LINE webhook event for audit.
- **ConversationView** — View projection used by the monitoring UI.
- **MessageConsumer** — Consumer that subscribes to `MessageQueue` events and starts a `ConversationWorkflow` per inbound message.
- **MessageSimulator** — TimedAction that drips sample LINE messages every 90 s so the App UI is never empty.
- **StaleConversationMonitor** — TimedAction that marks conversations stuck in `AWAITING_CONFIRMATION` past 10 minutes as `EXPIRED`.
- **LineWebhookEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the simulator message set or disable it.
- `SPEC.md §5` — add fields to `CalendarActionParams` (e.g., `attendees`) or `GmailActionParams` (e.g., `ccList`).
- `prompts/planner.md` — restrict the planner to a single domain (e.g., calendar only) or add a third executor.
- `eval-matrix.yaml` — add an `eval-event` control to score intent classification accuracy.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Send "Schedule a meeting with Alice tomorrow at 3 PM" → planner routes to Calendar executor → event created → LINE reply confirms with event details.
2. Send "Email Bob the project update" → guardrail passes → confirmation gate fires → user approves → email sent → LINE reply confirms.
3. Send a message instructing the agent to email an external address not in the allow-list → guardrail blocks the dispatch → LINE reply explains the action was not permitted.
4. Send an email-compose request and let the confirmation window expire → conversation moves to `EXPIRED` → no email sent → user receives a timeout notice.

## License

Apache 2.0.
