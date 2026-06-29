# Akka Sample: Support Agent with HITL

A customer support ticket is triaged by an agent, paused at a human review gate when sensitive actions are required, and resolved by a second agent only after an operator makes a decision.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider during generation (no key needed).
- Host software: None. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./human-in-loop-gate.cx-support.support-hitl-durable  ~/my-projects/support-hitl
cd ~/my-projects/support-hitl
```

(Optional) Edit `SPEC.md` to change the system name or model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- `TriageAgent` (AutonomousAgent) — classifies a support ticket, extracts a sanitized summary, and returns a typed `TicketTriage{category, priority, summary}`.
- `ResolutionAgent` (AutonomousAgent) — drafts a customer-facing response for an approved ticket, returns a typed `TicketResolution{response, resolvedAt}`.
- `SupportWorkflow` (Workflow) — a 3-task graph: triage → await operator decision → resolve.
- `TicketEntity` (EventSourcedEntity) — the ticket lifecycle and its events.
- `TicketsView` (View) — a read model the UI queries and streams over SSE.
- `SupportEndpoint` + `AppEndpoint` (HttpEndpoints) — the REST/SSE surface and the static UI.

## Customise before generating

- `SPEC.md` Section 1 — the system name.
- `SPEC.md` Section 11 — the model provider and HTTP port.
- `prompts/triage-agent.md` — the triage classification and sanitization instructions.

## What gets validated

- Submitting a ticket triggers triage; the ticket appears in `TRIAGED` with a non-empty sanitized summary.
- Approving a `TRIAGED` ticket drives it to `RESOLVED` with a non-null customer response within ~30 s.
- Rejecting a `TRIAGED` ticket drives it to terminal `ESCALATED` with the operator's reason shown.
- The resolution step never runs unless the ticket is `APPROVED`.
- PII patterns are absent from the persisted summary field.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
