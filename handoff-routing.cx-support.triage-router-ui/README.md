# Akka Sample: Triage Agent (UX)

A `RouterAgent` classifies an inbound customer support message, then hands the same `RESOLVE` task off to a `BillingHandler` or `ProductHandler` that owns the resolution end-to-end. A self-contained embedded UI lets you watch messages flow through triage and routing in real time. Demonstrates the **handoff-routing** coordination pattern with a before-agent-response guardrail on every specialist draft — the only governance control specified for this baseline.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host-software requirement: **None** — this blueprint runs out of the box. The inbound message stream and the UI are both modeled inside the service.

## Generate the system

```sh
cp -r ./handoff-routing.cx-support.triage-router-ui  ~/my-projects/triage-router-ui
cd ~/my-projects/triage-router-ui
```

(Optional) Edit `SPEC.md` to add routing categories (e.g. `ACCOUNT`, `SHIPPING`) and matching handler agents, or to point `MessageSimulator` at a real queue source.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **MessageSimulator** — TimedAction firing every 30 s that drips canned support messages from a JSONL file into `MessageQueue`.
- **MessageQueue** — EventSourcedEntity append-only log of every inbound message (pre-sanitization audit record).
- **RouterAgent** — typed Agent that classifies the message into `BILLING`, `PRODUCT`, or `UNCLEAR`.
- **BillingHandler** — AutonomousAgent that owns the `RESOLVE` task for billing messages (charges, refunds, plan changes).
- **ProductHandler** — AutonomousAgent that owns the `RESOLVE` task for product messages (usage questions, errors, integration help).
- **DraftGuardrail** — typed Agent used as a before-agent-response check on each specialist's draft reply.
- **TriageWorkflow** — Workflow per message: route → resolve → guardrail check → publish.
- **MessageEntity** — EventSourcedEntity holding each message's lifecycle.
- **MessageView + TriageEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — add routing categories and matching handler agents.
- `SPEC.md §5` — extend the `Message` record with deployer-specific fields (`customerTier`, `productLine`, `slaDeadline`).
- `prompts/router-agent.md` — tighten the classifier rules (confidence thresholds, category-specific keywords).
- `prompts/billing-handler.md` / `prompts/product-handler.md` — encode brand voice, refund authority limits, escalation triggers.
- `eval-matrix.yaml` — add additional guardrail rules or a sanitizer if your deployment handles PII.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Simulator drips a billing-flavoured message → triaged as `BILLING`, handed off to `BillingHandler`, resolution published.
2. Simulator drips a product message → triaged as `PRODUCT`, handed off to `ProductHandler`, resolution published.
3. An ambiguous message triages as `UNCLEAR` and the workflow terminates in `ESCALATED` without invoking any specialist.
4. A specialist draft that violates the before-agent-response guardrail (e.g. promises a specific refund timeline not in policy) is blocked; the message lands in `BLOCKED` for operator review.
5. An operator can unblock a blocked message via the UI, which publishes the draft and records the override note.

## License

Apache 2.0.
