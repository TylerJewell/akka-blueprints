# Akka Sample: SK Multi-Agent Handoff

A `TriageAgent` classifies an incoming conversation turn by intent, then hands the same `RESOLVE` task to an `AccountSpecialist`, `ProductSpecialist`, or `ReturnSpecialist` that owns the resolution end-to-end. Demonstrates the **handoff-routing** coordination pattern wired with two governance mechanisms: a before-agent-invocation guardrail that validates the routing decision before the specialist is called, and a before-agent-response guardrail that checks the specialist's draft before it reaches the user.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host-software requirement: **None** — this blueprint runs out of the box. The inbound conversation stream and the outbound resolution surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./handoff-routing.cx-support.triage-handoff-routing  ~/my-projects/triage-handoff
cd ~/my-projects/triage-handoff
```

(Optional) Edit `SPEC.md` to point `ConversationSimulator` at a real chat source (Intercom, Zendesk, Freshdesk) or to change the intent categories.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ConversationSimulator** — TimedAction firing every 30 s that drips canned conversation turns from a JSONL file into `ConversationQueue`.
- **ConversationQueue** — EventSourcedEntity append-only log of every inbound turn (audit before routing).
- **IntentClassifier** — Consumer that normalises the raw turn and prepares a `ClassifiedTurn` before any LLM call.
- **TriageAgent** — typed Agent that classifies the turn into `ACCOUNT`, `PRODUCT`, `RETURNS`, or `UNCLEAR`.
- **RoutingGuardrail** — typed Agent used as a before-agent-invocation guardrail; validates the routing decision before the specialist is invoked.
- **AccountSpecialist** — AutonomousAgent that owns the `RESOLVE` task for account-related turns (login, billing questions, subscription management).
- **ProductSpecialist** — AutonomousAgent that owns the `RESOLVE` task for product-related turns (how-to, features, integrations).
- **ReturnSpecialist** — AutonomousAgent that owns the `RESOLVE` task for returns-related turns (return requests, refund status, exchange).
- **ResponseGuardrail** — typed Agent used as a before-agent-response guardrail; checks the specialist's draft before it reaches the user.
- **HandoffWorkflow** — Workflow per conversation: classify → routing-check → handoff → specialist → response-check → publish.
- **ConversationEntity** — EventSourcedEntity holding each conversation turn's lifecycle.
- **ConversationView + HandoffEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the intent taxonomy (add `SHIPPING`, `LOYALTY`, etc.) and add the matching specialist agent.
- `SPEC.md §5` — extend the `Conversation` record with deployer-specific fields (`customerId`, `channelId`, `languageCode`).
- `prompts/triage-agent.md` — narrow the classifier rules (default-to-escalation thresholds, intent-specific keywords).
- `prompts/account-specialist.md` / `prompts/product-specialist.md` / `prompts/return-specialist.md` — encode brand voice, authority limits, escalation triggers.
- `eval-matrix.yaml` — swap the routing guardrail rubric for a deployer-specific policy set.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Simulator drips an account-intent turn → routing guardrail passes → handed to `AccountSpecialist` → response guardrail passes → published.
2. Simulator drips a product-intent turn → routed to `ProductSpecialist` → response guardrail passes → published.
3. A returns-intent turn → routed to `ReturnSpecialist` → response guardrail passes → published.
4. A routing guardrail failure (routing decision outside policy confidence threshold) blocks the handoff; the turn lands in `ROUTING_BLOCKED` for human review.
5. A specialist draft that violates the response rubric is blocked; the turn lands in `RESPONSE_BLOCKED` for human review.
6. An ambiguous turn triages as `UNCLEAR` and terminates in `ESCALATED` without ever calling a specialist.

## License

Apache 2.0.
