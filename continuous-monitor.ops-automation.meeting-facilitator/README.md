# Akka Sample: Teams Facilitator

A continuous background worker monitors active meeting sessions, sanitizes transcripts to remove PII, generates structured meeting notes and chat summaries with an AI, and gates every published summary behind a before-agent-response guardrail before it is shared with participants. Demonstrates the **continuous-monitor** coordination pattern wired with two governance mechanisms (PII sanitizer, before-agent-response guardrail).

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Integration-tier host software: **None** — meetings are simulated in-process; no Teams API credentials required.

## Generate the system

```sh
cp -r ./continuous-monitor.ops-automation.meeting-facilitator  ~/my-projects/meeting-facilitator
cd ~/my-projects/meeting-facilitator
```

(Optional) Edit `SPEC.md` to point the `MeetingSessionPoller` at a real transcript source (Graph API, Webhook) or to keep the in-memory simulator.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **MeetingSessionPoller** — TimedAction firing every 20 s that drips simulated transcript segments into `MeetingSessionQueue`.
- **TranscriptSanitizer** — Consumer that redacts PII from raw transcript segments before they reach any AI.
- **NotesSummaryAgent** — Agent (typed) that produces structured meeting notes from a sanitized segment batch.
- **ChatSummaryAgent** — Agent (typed) that distils the chat thread into a concise recap.
- **MeetingFacilitatorWorkflow** — per-session orchestration: sanitize → summarize notes → summarize chat → guardrail check → publish.
- **MeetingSessionEntity** — EventSourcedEntity holding each session's lifecycle (active → notes-generated → published / suppressed).
- **MeetingSessionView + MeetingEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **EvalRunner** — TimedAction running every 30 minutes; samples published summaries, scores them.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the simulated transcript source for a real Graph API webhook.
- `SPEC.md §5` — add org-specific fields (`meetingType`, `confidentialityLabel`) to `MeetingSession`.
- `prompts/notes-summary.md` — tune the note-taking style (bullet format, verbosity, template headers).
- `prompts/chat-summary.md` — scope the chat summariser to specific channels or thread types.
- `eval-matrix.yaml` — wire a real PII redactor (e.g., Microsoft Presidio) under the sanitizer mechanism's implementation.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A simulated transcript segment arrives → it is sanitized → notes drafted → guardrail passes → summary published.
2. A segment containing PII names triggers redaction; the LLM never sees raw names.
3. A guardrail-flagged summary is suppressed rather than published.
4. Eval scoring runs every 30 minutes and surfaces a quality score on published summaries.

## License

Apache 2.0.
