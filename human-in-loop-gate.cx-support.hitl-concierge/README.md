# Akka Sample: HITL Concierge

A customer request is triaged by an agent, paused at a human approval gate, and fulfilled by a second agent only after a support specialist approves the proposed resolution.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider during generation (no key needed).
- Host software: None. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./human-in-loop-gate.cx-support.hitl-concierge  ~/my-projects/hitl-concierge
cd ~/my-projects/hitl-concierge
```

(Optional) Edit `SPEC.md` to change the system name or model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- `TriageAgent` (AutonomousAgent) — analyses a customer request and returns a typed `TriageResult{summary, proposedResolution, urgency}`.
- `FulfillmentAgent` (AutonomousAgent) — executes an approved resolution and returns a typed `FulfilledCase{confirmationId, fulfilledAt}`.
- `ConciergeWorkflow` (Workflow) — a 3-task graph: triage → await specialist approval → fulfil.
- `CaseEntity` (EventSourcedEntity) — the support case lifecycle and its events.
- `CasesView` (View) — a read model the UI queries and streams over SSE.
- `ConciergeEndpoint` + `AppEndpoint` (HttpEndpoints) — the REST/SSE surface and the static UI.

## Customise before generating

- `SPEC.md` Section 1 — the system name.
- `SPEC.md` Section 11 — the model provider and HTTP port.
- `prompts/triage-agent.md` — the triage instructions and escalation rules.

## What gets validated

- Submitting a customer request produces a case in `TRIAGED` with a non-empty proposed resolution.
- Approving a `TRIAGED` case drives it to `FULFILLED` with a non-null confirmation id.
- Declining a `TRIAGED` case drives it to terminal `DECLINED` with the specialist's reason shown.
- The fulfilment step never runs unless the case is `APPROVED`.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
