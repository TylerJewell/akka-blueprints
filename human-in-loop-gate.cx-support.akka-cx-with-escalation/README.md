# Akka Sample: OpenAI Agents Customer Service with Escalation

A customer service agent handles inbound support conversations, holds durable state across turns, and escalates to a human agent when it cannot resolve the issue or when the customer requests it.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider during generation (no key needed).
- Host software: None. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./human-in-loop-gate.cx-support.akka-cx-with-escalation  ~/my-projects/akka-cx-with-escalation
cd ~/my-projects/akka-cx-with-escalation
```

(Optional) Edit `SPEC.md` to change the system name or model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- `SupportAgent` (AutonomousAgent) — handles customer messages, decides whether to resolve or escalate, returns a typed `AgentReply{message, action}`.
- `EscalationAgent` (AutonomousAgent) — drafts a handoff summary for a human agent and returns a typed `HandoffSummary{summary, priority}`.
- `SupportWorkflow` (Workflow) — drives the conversation: receive → respond → await escalation decision → optionally escalate.
- `ConversationEntity` (EventSourcedEntity) — holds full conversation state and lifecycle events.
- `ConversationsView` (View) — a read model the UI queries and streams over SSE.
- `SupportEndpoint` + `AppEndpoint` (HttpEndpoints) — the REST/SSE surface and the static UI.

## Customise before generating

- `SPEC.md` Section 1 — the system name.
- `SPEC.md` Section 11 — the model provider and HTTP port.
- `prompts/support-agent.md` — the triage instructions and escalation criteria.

## What gets validated

- Submitting a customer message opens a conversation that appears in `ACTIVE` with a non-empty agent reply.
- Sending an escalation signal transitions the conversation to `ESCALATING`, then `ESCALATED`, with a non-null `handoffSummary`.
- A resolved conversation reaches terminal `RESOLVED` without going through `ESCALATED`.
- The PII sanitizer strips recognized personal data before it is recorded on the entity.
- The before-agent-response guardrail blocks replies that fail a basic tone and content check.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
