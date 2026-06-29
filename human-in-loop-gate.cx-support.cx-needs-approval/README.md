# Akka Sample: AssistLoop Customer Support

A customer submits a support ticket. `TriageAgent` drafts a resolution response. The workflow pauses at a human approval gate. A support supervisor approves or rejects through the API. On approval, `ResolutionAgent` delivers the response to the customer and marks the ticket resolved.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider during generation (no key needed).
- Host software: None. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./human-in-loop-gate.cx-support.cx-needs-approval  ~/my-projects/assistloop-support
cd ~/my-projects/assistloop-support
```

(Optional) Edit `SPEC.md` to change the system name or model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- `TriageAgent` (AutonomousAgent) — analyzes an incoming ticket and drafts a support response, returns a typed `DraftResponse{summary, body}`.
- `ResolutionAgent` (AutonomousAgent) — delivers the approved response to the customer, returns a typed `DeliveredResolution{confirmationId, deliveredAt}`.
- `SupportWorkflow` (Workflow) — a 3-task graph: triage → await approval → resolve.
- `TicketEntity` (EventSourcedEntity) — the ticket lifecycle and its events.
- `TicketsView` (View) — a read model the UI queries and streams over SSE.
- `SupportEndpoint` + `AppEndpoint` (HttpEndpoints) — the REST/SSE surface and the static UI.

## Customise before generating

- `SPEC.md` Section 1 — the system name.
- `SPEC.md` Section 11 — the model provider and HTTP port.
- `prompts/triage-agent.md` — the triage and drafting instructions.

## What gets validated

- Submitting a ticket drafts a response that appears in `TRIAGED` with non-empty `responseBody`.
- Approving a `TRIAGED` ticket drives it to `RESOLVED` with a non-null `confirmationId`.
- Rejecting a `TRIAGED` ticket drives it to terminal `REJECTED` with the reason shown.
- The resolution step never runs unless the ticket is `APPROVED`.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
