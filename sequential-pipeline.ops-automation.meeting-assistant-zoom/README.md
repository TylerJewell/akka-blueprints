# Akka Sample: Zoom AI Meeting Assistant

A single `MeetingAgent` walks a Zoom meeting transcript through three task phases — **TRANSCRIBE → SUMMARIZE → DISPATCH** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a small set of phase-specific tools. The user submits a meeting recording and receives a structured `MeetingPackage` containing a summary, action items, and scheduled follow-ups.

Demonstrates the **sequential-pipeline** coordination pattern with two governance mechanisms: a `before-tool-call` guardrail that checks scope before any task or calendar write executes, and a `pii-sanitizer` that strips participant PII from transcript text before the LLM processes it.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every transcribe / summarize / dispatch tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.ops-automation.meeting-assistant-zoom  ~/my-projects/meeting-assistant-zoom
cd ~/my-projects/meeting-assistant-zoom
```

(Optional) Edit `SPEC.md` to change the seeded meeting set, the PII redaction patterns, or the action-item extraction heuristics.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **MeetingAgent** — one AutonomousAgent declaring three Task constants (`TRANSCRIBE_MEETING`, `SUMMARIZE_MEETING`, `DISPATCH_FOLLOWUPS`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **MeetingPipelineWorkflow** — runs `transcribeStep → summarizeStep → dispatchStep → evalStep`. Each step calls `runSingleTask` and writes the typed result back onto `MeetingEntity` before the next step starts.
- **MeetingEntity** — an EventSourcedEntity holding the per-meeting lifecycle (`TranscriptProduced`, `SummaryProduced`, `FollowUpsDispatched`, `EvaluationScored`).
- **TranscribeTools / SummarizeTools / DispatchTools** — three function-tool classes registered on the agent, one per phase. The `before-tool-call` guardrail enforces that each tool is only callable in its own phase and that dispatch writes are within scope.
- **ScopeGuardrail** — the runtime check backing the write-scope contract. Any calendar or task write whose attendee list is not a subset of the recorded meeting participants is rejected before the write executes.
- **PiiSanitizer** — runs before the TRANSCRIBE task; redacts names, email addresses, and phone numbers from the raw transcript before the text enters the LLM context.
- **ActionItemScorer** — deterministic, rule-based on-decision evaluator that runs immediately after `FollowUpsDispatched` and emits a 1–5 score for action-item completeness and assignee coverage.
- **MeetingView + MeetingEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded meeting set under `src/main/resources/sample-events/meetings.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/meeting-agent.md` — narrow the agent's role (e.g., constrain it to engineering standups, to sales call follow-ups, or to board meetings) by tightening the system prompt and renaming the typed records.
- `SPEC.md §5` — extend the typed outputs (`Transcript`, `MeetingSummary`, `MeetingPackage`) with domain-specific fields. The phase-gating guardrail does not need editing — it checks recorded-phase preconditions, not field shapes.
- `eval-matrix.yaml` — wire a real assignee-resolution tool (replace the deterministic stub with a directory lookup against your org's identity provider) by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a meeting recording → `TRANSCRIBE` runs → `SUMMARIZE` runs → `DISPATCH` runs → a typed `MeetingPackage` with a summary and action items lands in the UI within ~90 s. Every transition is visible in real time.
2. The guardrail blocks a dispatch-phase tool called before `SummaryProduced` has been recorded — the workflow records the rejection event, the agent retries in-phase, and the pipeline completes correctly.
3. Every `MeetingPackage` emitted has an on-decision eval score visible on the same UI card; packages whose action items have no assignee receive a score ≤ 2 and are flagged.
4. PII redaction fires before the TRANSCRIBE task; the LLM never sees raw participant names or contact details — the sanitizer output is stored on the entity alongside the redaction audit.

## License

Apache 2.0.
