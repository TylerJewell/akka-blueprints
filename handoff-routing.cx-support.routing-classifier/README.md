# Akka Sample: Routing Workflow

A `ClassifierAgent` reads an incoming support ticket and routes it to one of three specialist follow-up agents — `BillingHandler`, `TechnicalHandler`, or `AccountHandler` — that each own the response end-to-end. Demonstrates the **handoff-routing** coordination pattern with two governance mechanisms: a PII sanitizer that runs before any LLM call, and a before-agent-response guardrail that checks every draft reply before it reaches the customer.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host-software requirement: **None** — this blueprint runs out of the box. The inbound ticket stream and the outbound reply surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./handoff-routing.cx-support.routing-classifier  ~/my-projects/routing-workflow
cd ~/my-projects/routing-workflow
```

(Optional) Edit `SPEC.md` to change the routing taxonomy (add `RETURNS`, `FRAUD`, etc.) and add the matching handler agent.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TicketSimulator** — TimedAction firing every 30 s that drips canned support tickets from a JSONL file into `TicketQueue`.
- **TicketQueue** — EventSourcedEntity append-only log of every inbound ticket (audit before redaction).
- **PiiSanitizer** — Consumer that redacts emails, phone numbers, card numbers, and account ids before any LLM call.
- **ClassifierAgent** — typed Agent that routes the sanitized ticket to `BILLING`, `TECHNICAL`, `ACCOUNT`, or `UNCLEAR`.
- **BillingHandler** — AutonomousAgent that owns the `RESPOND` task for billing tickets (charges, refunds, plan changes).
- **TechnicalHandler** — AutonomousAgent that owns the `RESPOND` task for technical tickets (errors, integrations, configuration).
- **AccountHandler** — AutonomousAgent that owns the `RESPOND` task for account-management tickets (credentials, upgrades, cancellations).
- **ResponseGuardrail** — typed Agent used as a before-agent-response check: validates every handler draft against a policy rubric before publishing.
- **RoutingWorkflow** — Workflow per ticket: sanitize → classify → route → handle → guardrail → publish.
- **CaseEntity** — EventSourcedEntity holding each ticket's lifecycle.
- **CaseView + RoutingEndpoint + AppEndpoint** — read model + REST/SSE + static UI.

A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the routing taxonomy (add `RETURNS`, `FRAUD`) and add the matching handler agent.
- `SPEC.md §5` — extend the `Case` record with deployer-specific fields (`customerTier`, `contractId`, `slaDeadlineAt`).
- `prompts/classifier-agent.md` — adjust category boundaries and ambiguity thresholds.
- `prompts/billing-handler.md`, `prompts/technical-handler.md`, `prompts/account-handler.md` — encode brand voice, refund authority, and escalation triggers.
- `eval-matrix.yaml` — swap the in-process PII regex for a real redaction service.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Simulator drops a billing-flavoured ticket → sanitized, classified `BILLING`, handed off to `BillingHandler`, guardrail passes, reply published.
2. Simulator drops a technical ticket → classified `TECHNICAL`, handled by `TechnicalHandler`, guardrail passes, reply published.
3. Simulator drops an account-management ticket → classified `ACCOUNT`, handled by `AccountHandler`, guardrail passes, reply published.
4. An ambiguous ticket classifies `UNCLEAR` and terminates in `ESCALATED` without invoking any handler.
5. A handler draft that violates the guardrail rubric (e.g., promises same-day resolution outside SLA) is blocked; the ticket sits in `BLOCKED` for operator review.

## License

Apache 2.0.
