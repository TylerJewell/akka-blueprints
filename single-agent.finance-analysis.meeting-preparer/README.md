# Akka Sample: Meeting Preparer

A single meeting-preparer agent assembles a structured client or counterparty brief ahead of a scheduled call. It pulls contact history from CRM, recent SEC filings or financial highlights, and live news, runs PII sanitization before the LLM ever sees the data, and returns a ready-to-read brief: executive summary, key talking points, risk flags, financial highlights, and recent news.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: a PII sanitizer that runs inside a Consumer before the agent call, so the model never sees raw contact identifiers.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the CRM and filings data live in-process as seeded JSONL, and the agent's tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.finance-analysis.meeting-preparer  ~/my-projects/meeting-preparer
cd ~/my-projects/meeting-preparer
```

(Optional) Edit `SPEC.md` to point at a different data source (e.g., swap the seeded CRM JSONL for a live CRM integration URL, or extend the brief template with a custom section for your firm's risk taxonomy).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **BriefingAgent** — an AutonomousAgent that accepts a meeting request and returns a typed `MeetingBrief`.
- **BriefingWorkflow** — orchestrates sanitize-wait → brief → eval per submitted meeting request.
- **BriefingEntity** — an EventSourcedEntity holding the per-brief lifecycle.
- **ContactSanitizer** — a Consumer that subscribes to `MeetingRequested` events, redacts PII from CRM contact data, and emits `ContactDataSanitized` back to the entity.
- **BriefingView + BriefingEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded counterparty profiles for your own (the JSONL file under `src/main/resources/sample-events/counterparties.jsonl` after generation).
- `SPEC.md §5` — extend `MeetingBrief` with firm-specific fields (e.g., `dealStage`, `regulatoryFlags`, `jurisdictionNotes`).
- `prompts/briefing-agent.md` — narrow the agent's role (an investment bank would constrain it to public-market data only; a corporate-development team would add M&A-context sections).
- `eval-matrix.yaml` — wire a real PII redactor (e.g., a presidio-style library) by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a meeting request → contact data is sanitized → brief is generated → the brief appears in the UI with all sections populated.
2. PII strings in the CRM payload never appear in the LLM call log; only the redacted form does.
3. A brief whose talking points contain no supporting evidence receives a low eval score, flagging it for human review before the call.
4. A live brief auto-updates in the UI via SSE as each pipeline stage completes.

## License

Apache 2.0.
