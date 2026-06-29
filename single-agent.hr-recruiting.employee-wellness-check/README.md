# Akka Sample: Wellness Check Agent

A single wellness-check agent sends periodic check-in messages to employees over configured channels, collects morale responses, redacts special-category wellness data before any model call, and routes crisis signals immediately to a human. Aggregate morale trends are surfaced on the App UI tab and held for a human-on-the-loop surveillance review.

Demonstrates the **single-agent** coordination pattern wired with three governance mechanisms: a special-category data sanitizer that strips identifiable health and mental-health markers before the agent processes any response, a `before-agent-response` guardrail that intercepts harmful or crisis-grade outputs and redirects them to a human escalation path, and a post-market-surveillance human-on-the-loop review of aggregated morale signals for workforce risk.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — channel delivery is simulated in-process and the agent's tool calls are stubbed.

## Generate the system

```sh
cp -r ./single-agent.hr-recruiting.employee-wellness-check  ~/my-projects/wellness-check
cd ~/my-projects/wellness-check
```

(Optional) Edit `SPEC.md` to change the seeded question library (e.g., swap the standard 5-question morale pulse for a burnout-specific instrument or a post-reorg sentiment survey).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **WellnessCheckAgent** — an AutonomousAgent that receives an employee's check-in response (passed as a task attachment, never as inline prompt text) and returns a typed `CheckInAnalysis` classifying morale level and flagging any crisis signals.
- **CheckInWorkflow** — orchestrates sanitize-wait → analyse → surveillance per submitted response.
- **CheckInEntity** — an EventSourcedEntity holding the per-check-in lifecycle.
- **ResponseSanitizer** — a Consumer that subscribes to `ResponseReceived` events, redacts special-category wellness identifiers into a `SanitizedResponse`, and emits `ResponseSanitized` back to the entity.
- **CheckInView + CheckInEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded question library for your own (the JSONL file under `src/main/resources/sample-events/questions.jsonl` after generation).
- `SPEC.md §5` — extend `CheckInAnalysis` with organisation-specific fields (e.g., `teamId`, `officeLocation`, `workloadRating`).
- `prompts/wellness-check-agent.md` — narrow the agent's classification role for your population (a crisis-line deployer would tighten crisis detection thresholds; a return-to-office deployer would add commute-sentiment categories).
- `eval-matrix.yaml` — wire a real special-category redactor by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. An HR coordinator schedules a check-in campaign → employees receive questions → responses are sanitized → the agent classifies morale → the summary appears on the UI dashboard.
2. The agent's first response on a check-in contains an out-of-enum morale level — the `before-agent-response` guardrail rejects it, the agent retries, and a valid classification lands.
3. A response containing explicit crisis language triggers the crisis-escalation path and is never surfaced as a standard morale result.
4. Wellness text submitted in the raw response never appears in the LLM call log; only the redacted form does.

## License

Apache 2.0.
