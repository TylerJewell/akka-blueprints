# Akka Sample: Inbox Watcher

A continuous background worker polls a simulated inbox, redacts PII from incoming messages, generates draft replies with an AI, and holds every draft for human approval before sending. Demonstrates the **continuous-monitor** coordination pattern wired with four governance mechanisms (PII sanitizer, before-tool-call guardrail, HITL approve-to-send, periodic eval).

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./continuous-monitor.cx-support.inbox-watcher  ~/my-projects/inbox-watcher
cd ~/my-projects/inbox-watcher
```

(Optional) Edit `SPEC.md` to point the InboxPoller at a real inbox source (Gmail, IMAP) or to keep the in-memory simulator.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **InboxPoller** — TimedAction firing every 15 s that pulls new messages from a simulated inbox source.
- **PiiSanitizer** — Consumer that redacts PII fields before the message is shown to the AI.
- **InboxFilterAgent** — Agent that classifies the message into `auto-drafted`, `escalate`, or `ignore`.
- **DraftReplyAgent** — AutonomousAgent that drafts a reply for `auto-drafted` messages.
- **InboxMessageEntity** — EventSourcedEntity holding each message's lifecycle (received → sanitized → classified → drafted → approved/rejected → sent).
- **InboxView + InboxEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **EvalRunner** — TimedAction running every 30 minutes; samples sent replies, scores them.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the simulated inbox for a real source.
- `SPEC.md §5` — extend the `InboxMessage` record with org-specific fields (`customerTier`, `slaDeadline`, etc.).
- `prompts/inbox-filter.md` — narrow the classifier to a specific support category.
- `eval-matrix.yaml` — wire a real PII redactor (e.g., Microsoft Presidio) by adding it under the sanitizer mechanism's implementation.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A simulated message arrives → it is sanitized → classified → drafted → held for approval.
2. A human clicks Approve → the message transitions to SENT (in simulation, no real outbound).
3. A human clicks Reject with a reason → the message transitions to DISMISSED.
4. Eval scoring runs every 30 minutes and surfaces a score on sent messages.

## License

Apache 2.0.
