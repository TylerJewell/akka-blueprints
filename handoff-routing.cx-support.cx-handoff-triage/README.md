# Akka Sample: Core Streaming Handoffs Customer Service

A `TriageAgent` classifies an inbound customer message, then hands the same `RESOLVE` task off to a `SalesSpecialist` or `IssuesRepairsSpecialist` that owns the response end-to-end. Demonstrates the **handoff-routing** coordination pattern wired with three governance mechanisms: a before-agent-response guardrail on every specialist draft, a before-tool-call guardrail on refund and order-fulfillment tool calls, and a PII sanitizer that redacts customer contact and order data before any LLM sees it. Conversation history is persisted per session and streamed as Server-Sent Events.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host-software requirement: **None** — this blueprint runs out of the box. The inbound chat stream and the conversation history store are both modeled inside the service.

## Generate the system

```sh
cp -r ./handoff-routing.cx-support.cx-handoff-triage  ~/my-projects/cx-handoff-triage
cd ~/my-projects/cx-handoff-triage
```

(Optional) Edit `SPEC.md` to point `ConversationSimulator` at a real channel source (LiveChat, Intercom, Zendesk Chat) or to add routing categories beyond `SALES` and `ISSUES_REPAIRS`.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ConversationSimulator** — TimedAction firing every 30 s that drips canned customer conversations from a JSONL file into `ConversationQueue`.
- **ConversationQueue** — EventSourcedEntity append-only log of every inbound message (audit before redaction).
- **PiiSanitizer** — Consumer that redacts emails, phone numbers, names, and order ids before any LLM call.
- **TriageAgent** — typed Agent that classifies the sanitized opening message into `SALES`, `ISSUES_REPAIRS`, or `UNCLEAR`.
- **SalesSpecialist** — AutonomousAgent that owns the `RESOLVE` task for sales conversations (pricing, upgrades, product questions, order placement).
- **IssuesRepairsSpecialist** — AutonomousAgent that owns the `RESOLVE` task for issues and repairs conversations (defects, returns, repair status, replacement parts).
- **ResponseGuardrail** — typed Agent enforcing the before-agent-response check on every specialist draft.
- **ToolCallGuardrail** — typed Agent enforcing the before-tool-call check on refund and order-fulfillment tool calls before execution.
- **ConversationWorkflow** — Workflow per conversation: sanitize-handoff → triage → route → resolve → guardrail → publish.
- **ConversationEntity** — EventSourcedEntity holding each conversation's lifecycle and history.
- **ConversationView + ChatEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — add routing categories (`RETURNS`, `BILLING`, `SHIPPING`) and matching specialist agents.
- `SPEC.md §5` — extend the `Conversation` record with deployer-specific fields (`customerId`, `productLine`, `loyaltyTier`).
- `prompts/triage-agent.md` — adjust classification rules (confidence thresholds, category keywords for your catalog).
- `prompts/sales-specialist.md` / `prompts/issues-repairs-specialist.md` — encode brand voice, discount authority limits, escalation triggers.
- `eval-matrix.yaml` — swap the in-process tool guardrail for a real policy-as-code sidecar if your compliance team requires it.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Simulator drips a sales-flavoured conversation → it is sanitized, triaged as `SALES`, handed off to `SalesSpecialist`, and a response is published.
2. Simulator drips an issues-and-repairs conversation → triaged as `ISSUES_REPAIRS`, handed off to `IssuesRepairsSpecialist`, and a response is published.
3. An ambiguous opening message triages as `UNCLEAR` and the workflow terminates in `ESCALATED` without invoking a specialist.
4. A specialist draft that violates the before-agent-response guardrail (e.g. promises a delivery window not in policy) is blocked; the conversation lands in `BLOCKED` for human review.
5. A refund or order-fulfillment tool call that fails the before-tool-call guardrail is cancelled; the specialist receives a policy-rejection signal and must choose an alternative action.

## License

Apache 2.0.
