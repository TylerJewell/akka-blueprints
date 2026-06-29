# Akka Sample: Multi-Agent Customer Router

An orchestrator agent classifies an incoming support ticket and routes it to one of three specialist agents — Billing, Technical, or Account Management — then persists the routing decision for audit. Demonstrates the **delegation-supervisor-workers** coordination pattern with a before-invocation guardrail on the routing decision and a post-resolution eval that scores whether the chosen specialist closed the ticket.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the inbound ticket stream and the specialist agents are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.cx-support.multi-agent-customer-router  ~/my-projects/customer-router
cd ~/my-projects/customer-router
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or routing categories.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **RouterAgent** — AutonomousAgent that classifies a ticket and selects the correct specialist.
- **BillingAgent** — AutonomousAgent that resolves billing enquiries.
- **TechnicalAgent** — AutonomousAgent that resolves technical support queries.
- **AccountAgent** — AutonomousAgent that handles account-management requests.
- **TicketWorkflow** — Workflow that runs the routing guardrail, delegates to the chosen specialist, and persists the outcome.
- **TicketEntity** — EventSourcedEntity holding the ticket's full lifecycle.
- **TicketQueueEntity** — EventSourcedEntity that logs each submitted ticket for replay/audit.
- **TicketView** — projection the UI streams via SSE.
- **TicketEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the ticket categories or the simulator drip rate.
- `SPEC.md §5` — adjust the `Ticket` record fields (e.g., add `priority`).
- `prompts/router.md` — add a fourth routing category without changing the workflow.
- `eval-matrix.yaml` — add a `before-tool-call` guardrail if you wire a real ticketing system.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a billing ticket → ticket progresses OPEN → ROUTING → IN_PROGRESS → RESOLVED; UI reflects each transition via SSE.
2. Router misclassifies a ticket (test fixture) → the before-invocation guardrail catches the mismatch before specialist handoff.
3. Specialist times out → ticket enters ESCALATED with a failure reason.
4. Eval sampler scores a resolved ticket → the ticket row shows a resolution score in the App UI.

## License

Apache 2.0.
