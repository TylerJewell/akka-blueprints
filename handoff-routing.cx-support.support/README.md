# Akka Sample: Support Handoff

A `TriageAgent` classifies an incoming support request, then hands the same `RESOLVE` task off to a `BillingSpecialist` or `TechnicalSpecialist` that owns the resolution end-to-end. Demonstrates the **handoff-routing** coordination pattern wired with three governance mechanisms (PII sanitizer before any LLM call, before-agent-response guardrail on the specialist's draft, and an on-decision eval that scores every triage call).

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host-software requirement: **None** — this blueprint runs out of the box. The inbound support stream and the outbound resolution surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./handoff-routing.cx-support.support  ~/my-projects/support-handoff
cd ~/my-projects/support-handoff
```

(Optional) Edit `SPEC.md` to point `RequestSimulator` at a real ticketing source (Zendesk, Intercom, Front) or to change the triage categories.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **RequestSimulator** — TimedAction firing every 30 s that drips canned support requests from a JSONL file into `RequestQueue`.
- **RequestQueue** — EventSourcedEntity append-only log of every inbound request (audit before redaction).
- **PiiSanitizer** — Consumer that redacts emails, phone numbers, card numbers, and order ids before any LLM call.
- **TriageAgent** — typed Agent that classifies the sanitized request into `BILLING`, `TECHNICAL`, or `UNCLEAR`.
- **BillingSpecialist** — AutonomousAgent that owns the `RESOLVE` task for billing tickets (charges, refunds, plan changes).
- **TechnicalSpecialist** — AutonomousAgent that owns the `RESOLVE` task for technical tickets (errors, integrations, how-tos).
- **TriageJudge** — typed Agent used by `TriageEvalScorer` to grade every triage decision against a 1–5 rubric.
- **SupportWorkflow** — Workflow per ticket: sanitize-handoff → triage → route → resolve → guardrail check → publish.
- **TicketEntity** — EventSourcedEntity holding each ticket's lifecycle.
- **TicketView + SupportEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **TriageEvalScorer** — Consumer that listens for `TriageDecided` events and writes an inline eval score.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the triage taxonomy (add `ACCOUNT`, `SHIPPING`, etc.) and add the matching specialist agent.
- `SPEC.md §5` — extend the `Ticket` record with deployer-specific fields (`customerTier`, `productSku`, `slaDeadline`).
- `prompts/triage-agent.md` — narrow the classifier rules (default-to-escalation thresholds, category-specific keywords).
- `prompts/billing-specialist.md` / `prompts/technical-specialist.md` — encode brand voice, refund authority limits, escalation triggers.
- `eval-matrix.yaml` — swap the in-process PII regex for a real redactor (e.g., a Presidio sidecar — make it Real-service-via-env-var if you do).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Simulator drips a billing-flavoured ticket → it is sanitized, triaged as `BILLING`, handed off to `BillingSpecialist`, and a resolution is published.
2. Simulator drips a technical ticket → it is sanitized, triaged as `TECHNICAL`, handed off to `TechnicalSpecialist`, and a resolution is published.
3. An ambiguous ticket triages as `UNCLEAR` and the workflow terminates in `ESCALATED` without ever calling a specialist.
4. A specialist draft that violates the before-agent-response guardrail (e.g. promises a specific refund timeline that isn't in policy) is blocked; the ticket lands in `BLOCKED` for human review.
5. The eval score (1–5) and rationale appear on every triaged ticket within a few seconds of the decision.

## License

Apache 2.0.
