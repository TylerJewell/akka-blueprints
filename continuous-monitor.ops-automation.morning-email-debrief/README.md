# Akka Sample: Morning Email Debrief

A scheduled background worker polls a mailbox on a configurable morning cadence, redacts PII from every email, summarises the batch with an AI agent, and surfaces a structured debrief digest for review. Demonstrates the **continuous-monitor** coordination pattern with two governance mechanisms (PII sanitizer, periodic eval).

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Integration-tier host software: **None** — the blueprint ships with an in-memory email simulator.

## Generate the system

```sh
cp -r ./continuous-monitor.ops-automation.morning-email-debrief  ~/my-projects/morning-email-debrief
cd ~/my-projects/morning-email-debrief
```

(Optional) Edit `SPEC.md §3` to point `MailboxPoller` at a real email source (Gmail API, IMAP) or keep the in-memory simulator.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **MailboxPoller** — TimedAction firing at a configurable morning schedule (default: every 60 s in dev mode) that pulls new emails from a simulated mailbox source.
- **EmailPiiSanitizer** — Consumer that redacts PII fields before any email content reaches the AI.
- **EmailSummarizerAgent** — Agent that produces a structured `DebriefEntry` for each email (one-line summary, priority, category).
- **DebriefAssemblerAgent** — AutonomousAgent that combines per-email entries into a `MorningDebrief` digest.
- **EmailEntity** — EventSourcedEntity holding each email's lifecycle (received → sanitized → summarised → assembled).
- **DebriefEntity** — EventSourcedEntity holding the daily debrief run (created → assembling → ready).
- **EmailView + DebriefView + DebriefEndpoint + AppEndpoint** — read models + REST/SSE + static UI.
- **EvalRunner** — TimedAction running every 60 minutes; samples completed debriefs, scores them on coverage and accuracy.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the simulated mailbox for a real email source (IMAP credentials, Gmail OAuth scope).
- `SPEC.md §5` — extend `ReceivedEmail` with org-specific fields (`senderTier`, `projectTag`, etc.).
- `prompts/email-summarizer.md` — narrow the summarizer to a specific email category or department.
- `prompts/debrief-assembler.md` — adjust the debrief format (executive bullet list, table, narrative).
- `eval-matrix.yaml` — wire a production PII redactor (e.g., a dedicated data-loss-prevention service) under the sanitizer mechanism's implementation.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A simulated email batch arrives → each email is sanitized → summarised → a debrief digest is assembled and visible in the UI.
2. A debrief run completes with no raw PII in the assembled output.
3. Eval scoring runs on a completed debrief and surfaces a coverage score in the UI.
4. A debrief in progress can be observed in real time via SSE.

## License

Apache 2.0.
