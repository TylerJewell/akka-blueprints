# Akka Sample: Support Multi-Agent

A `RouterAgent` classifies an incoming customer support request, then hands the same `RESOLVE` task off to a `BillingSpecialist`, `TechnicalSpecialist`, or `AccountSpecialist` that owns the resolution end-to-end. Demonstrates the **handoff-routing** coordination pattern wired with two governance mechanisms (PII sanitizer before any LLM call, and a before-agent-response guardrail on every specialist's draft).

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host-software requirement: **None** — this blueprint runs out of the box. The inbound support stream and the outbound resolution surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./handoff-routing.cx-support.support-router-team  ~/my-projects/support-router-team
cd ~/my-projects/support-router-team
```

(Optional) Edit `SPEC.md` to point `TicketSimulator` at a real ticketing source (Zendesk, Intercom, Front) or to change the routing categories.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TicketSimulator** — TimedAction firing every 30 s that drips canned support requests from a JSONL file into `TicketQueue`.
- **TicketQueue** — EventSourcedEntity append-only log of every inbound request (audit before redaction).
- **PiiSanitizer** — Consumer that redacts emails, phone numbers, card numbers, and account ids before any LLM call.
- **RouterAgent** — typed Agent that classifies the sanitized request into `BILLING`, `TECHNICAL`, `ACCOUNT`, or `UNCLEAR`.
- **BillingSpecialist** — AutonomousAgent that owns the `RESOLVE` task for billing tickets (charges, refunds, plan changes).
- **TechnicalSpecialist** — AutonomousAgent that owns the `RESOLVE` task for technical tickets (errors, integrations, how-tos).
- **AccountSpecialist** — AutonomousAgent that owns the `RESOLVE` task for account management tickets (access, profile, subscription settings).
- **DraftGuardrail** — typed Agent used as the before-agent-response check on every specialist's draft.
- **SupportWorkflow** — Workflow per ticket: sanitize → route → resolve → guardrail → publish.
- **CaseEntity** — EventSourcedEntity holding each case's lifecycle.
- **CaseView + SupportEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the routing taxonomy (add `SHIPPING`, `RETURNS`, etc.) and add the matching specialist agent.
- `SPEC.md §5` — extend the `Case` record with deployer-specific fields (`customerTier`, `productSku`, `slaDeadline`).
- `prompts/router-agent.md` — narrow the classifier rules (default-to-escalation thresholds, category-specific keywords).
- `prompts/billing-specialist.md` / `prompts/technical-specialist.md` / `prompts/account-specialist.md` — encode brand voice, authority limits, escalation triggers.
- `eval-matrix.yaml` — swap the in-process PII regex for a real redactor (e.g. a Presidio sidecar — make it Real-service-via-env-var if you do).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Simulator drips a billing-flavoured case → it is sanitized, routed to `BILLING`, handed off to `BillingSpecialist`, and a resolution is published.
2. Simulator drips a technical case → it is sanitized, routed to `TECHNICAL`, handed off to `TechnicalSpecialist`, and a resolution is published.
3. Simulator drips an account management case → it is sanitized, routed to `ACCOUNT`, handed off to `AccountSpecialist`, and a resolution is published.
4. An ambiguous case routes as `UNCLEAR` and the workflow terminates in `ESCALATED` without ever calling a specialist.
5. A specialist draft that violates the before-agent-response guardrail (e.g. promises a specific refund timeline that is not in policy) is blocked; the case lands in `BLOCKED` for human review.

## License

Apache 2.0.
