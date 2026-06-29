# Akka Sample: OpenAI Agents SDK Handoff Routing

A `TriageAgent` receives an inbound conversation turn, checks a routing guardrail before selecting a specialist, then hands the full conversation context to a `BillingSpecialist` or `TechnicalSpecialist` that owns resolution end-to-end. A PII sanitizer runs before any LLM call. The message-filter variant demonstrates how to strip sensitive fields from the context bundle passed across a handoff boundary.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host-software requirement: **None** — this blueprint runs out of the box. The inbound conversation stream and the outbound reply surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./handoff-routing.cx-support.akka-handoff-routing  ~/my-projects/akka-handoff-routing
cd ~/my-projects/akka-handoff-routing
```

(Optional) Edit `SPEC.md` to point `ConversationSimulator` at a real messaging source or to extend the routing categories.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ConversationSimulator** — TimedAction firing every 30 s that drips canned conversation turns from a JSONL file into `ConversationQueue`.
- **ConversationQueue** — EventSourcedEntity append-only log of every inbound turn (audit before redaction).
- **MessageFilter** — Consumer that redacts emails, phone numbers, card numbers, and account ids from the conversation payload before any LLM call.
- **RoutingGuardrail** — typed Agent invoked before the handoff decision: checks whether the routing context is safe to pass to a specialist.
- **TriageAgent** — typed Agent that classifies the filtered conversation into `BILLING`, `TECHNICAL`, or `UNCLEAR`.
- **BillingSpecialist** — AutonomousAgent that owns the `RESOLVE` task for billing conversations (charges, refunds, plan changes).
- **TechnicalSpecialist** — AutonomousAgent that owns the `RESOLVE` task for technical conversations (errors, integrations, how-tos).
- **HandoffJudge** — typed Agent used by `HandoffEvalScorer` to grade every routing decision against a 1–5 rubric.
- **HandoffWorkflow** — Workflow per conversation: filter → guardrail-check → triage → route → resolve → publish.
- **ConversationEntity** — EventSourcedEntity holding each conversation's lifecycle.
- **ConversationView + HandoffEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **HandoffEvalScorer** — Consumer that listens for `RoutingDecided` events and writes an inline eval score.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the routing taxonomy (add `ACCOUNT`, `SHIPPING`, etc.) and add the matching specialist agent.
- `SPEC.md §5` — extend the `Conversation` record with deployer-specific fields (`customerTier`, `productLine`, `slaDeadline`).
- `prompts/triage-agent.md` — narrow the classifier rules (ambiguity thresholds, category-specific signals).
- `prompts/billing-specialist.md` / `prompts/technical-specialist.md` — encode brand voice, refund authority limits, escalation triggers.
- `eval-matrix.yaml` — swap the in-process PII regex for a real redactor.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Simulator drips a billing-flavoured conversation → routing guardrail passes → triaged as `BILLING` → handed off to `BillingSpecialist` → reply published.
2. Simulator drips a technical conversation → triaged as `TECHNICAL` → handed off to `TechnicalSpecialist` → reply published.
3. An ambiguous conversation triages as `UNCLEAR` and the workflow terminates in `ESCALATED` without ever calling a specialist.
4. A routing context that fails the before-handoff guardrail (e.g. contains unredacted PII in the context bundle) is blocked before the specialist sees it; the conversation lands in `BLOCKED` for human review.
5. The eval score (1–5) and rationale appear on every routed conversation within a few seconds of the routing decision.

## License

Apache 2.0.
