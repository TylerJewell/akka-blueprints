# Akka Sample: Human-in-the-Loop Hiring

A candidate application is evaluated by two agents — one proposes a hiring decision, another proposes a meeting schedule — and both proposals wait at a human approval gate before any outcome is recorded.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider during generation (no key needed).
- Host software: None. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./human-in-loop-gate.hr-recruiting.hiring-hitl-gate  ~/my-projects/hiring-hitl-gate
cd ~/my-projects/hiring-hitl-gate
```

(Optional) Edit `SPEC.md` to change the system name or model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- `HiringDecisionProposer` (AutonomousAgent) — evaluates a candidate application and returns a typed `HiringProposal{recommendation, rationale}`.
- `MeetingProposer` (AutonomousAgent) — drafts a meeting invitation and returns a typed `MeetingProposal{subject, body, suggestedSlots}`.
- `HiringWorkflow` (Workflow) — a 3-task graph: propose decision → await validator approval → record outcome.
- `ApplicationEntity` (EventSourcedEntity) — the candidate application lifecycle and its events.
- `ApplicationsView` (View) — a read model the UI queries and streams over SSE.
- `HiringEndpoint` + `AppEndpoint` (HttpEndpoints) — the REST/SSE surface and the static UI.

## Customise before generating

- `SPEC.md` Section 1 — the system name.
- `SPEC.md` Section 11 — the model provider and HTTP port.
- `prompts/hiring-decision-proposer.md` — the evaluation criteria and recommendation format.
- `prompts/meeting-proposer.md` — the interview scheduling tone and slot format.

## What gets validated

- Submitting an application triggers both agents and the application appears in `PROPOSED` with non-empty proposal text.
- Approving a `PROPOSED` application drives it to `ACCEPTED` with the decision recorded.
- Rejecting a `PROPOSED` application drives it to terminal `DECLINED` with the reason shown.
- The outcome-recording step never runs unless the validator has approved.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
