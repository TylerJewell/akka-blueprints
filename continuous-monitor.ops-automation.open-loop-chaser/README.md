# Akka Sample: Intelligent Reminders

A continuous background worker scans meeting notes, messages, and documents for open action items, strips PII from the text before any model call, and sends targeted nudges to the responsible owner. Every nudge is gated by a before-tool-call guardrail so the system cannot dispatch communications to humans unless the item passes the guard. Demonstrates the **continuous-monitor** coordination pattern with two governance mechanisms (PII sanitizer, before-tool-call guardrail).

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Integration-tier host software: **None** — the system ships with a built-in source simulator; no external calendar, messaging, or document system is required to run it.

## Generate the system

```sh
cp -r ./continuous-monitor.ops-automation.open-loop-chaser  ~/my-projects/open-loop-chaser
cd ~/my-projects/open-loop-chaser
```

(Optional) Edit `SPEC.md` to connect the `ActionItemPoller` to a real source (calendar API, Slack, Confluence) instead of the in-memory simulator.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ActionItemPoller** — TimedAction firing every 20 s that pulls new action items from a simulated source (meeting notes, messages, documents).
- **PiiSanitizer** — Consumer that redacts PII fields from raw source text before the item is processed by the AI.
- **ActionItemExtractorAgent** — Agent (typed) that parses the sanitized text and extracts structured action items: owner, description, due date, source type.
- **NudgeComposerAgent** — AutonomousAgent that drafts a reminder message for each open item past its staleness threshold.
- **ActionItemEntity** — EventSourcedEntity holding each item's lifecycle (detected → sanitized → extracted → pending → nudged / closed / snoozed).
- **ActionItemView + ActionItemEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **StaleItemScanner** — TimedAction running every 10 minutes; finds items overdue for a nudge and queues them.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the simulated source for a real one (calendar events, Slack channel, Confluence pages).
- `SPEC.md §5` — extend the `ActionItem` record with org-specific fields (`projectCode`, `priority`, `escalationOwner`).
- `prompts/action-item-extractor.md` — narrow the extractor to a specific team's naming conventions.
- `eval-matrix.yaml` — add regulation anchors if your deployment context requires them.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A simulated source document is ingested → text is sanitized → action items are extracted → items appear in the UI.
2. A stale item triggers the nudge path; the nudge is held by the guardrail until the item has a confirmed owner.
3. An operator marks an item closed via the UI → the item transitions to CLOSED and no further nudges fire.
4. An operator snoozes an item → nudges are suppressed until the snooze window expires.

## License

Apache 2.0.
