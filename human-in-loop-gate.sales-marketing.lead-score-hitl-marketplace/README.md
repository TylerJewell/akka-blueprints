# Akka Sample: Lead Score Flow

A CRM lead is scored by an agent, paused at a human review gate when the score is ambiguous, and qualified or disqualified by a sales reviewer before any downstream action is taken.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider during generation (no key needed).
- Host software: None. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./human-in-loop-gate.sales-marketing.lead-score-hitl-marketplace  ~/my-projects/lead-score-flow
cd ~/my-projects/lead-score-flow
```

(Optional) Edit `SPEC.md` to change the system name or model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- `ScoringAgent` (AutonomousAgent) — scores a lead against ICP criteria, returns a typed `LeadScore{score, rationale, confidence}`.
- `QualificationAgent` (AutonomousAgent) — generates a qualification summary after human approval, returns a typed `QualificationSummary{verdict, nextSteps}`.
- `LeadScoringWorkflow` (Workflow) — a 3-task graph: score → await review → qualify.
- `LeadEntity` (EventSourcedEntity) — the lead lifecycle and its events.
- `LeadsView` (View) — a read model the UI queries and streams over SSE.
- `LeadScoringEndpoint` + `AppEndpoint` (HttpEndpoints) — the REST/SSE surface and the static UI.

## Customise before generating

- `SPEC.md` Section 1 — the system name.
- `SPEC.md` Section 11 — the model provider and HTTP port.
- `prompts/scoring-agent.md` — the ICP scoring criteria and confidence thresholds.

## What gets validated

- Submitting a lead profile triggers scoring and the lead appears in `SCORED` with a non-null score and rationale.
- Approving a `SCORED` lead drives it to `QUALIFIED` with a non-null qualification summary.
- Rejecting a `SCORED` lead drives it to terminal `DISQUALIFIED` with the reason shown.
- The qualification step never runs unless the lead is `APPROVED`.
- Eval events are emitted on each scoring decision and appear in the event log.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
