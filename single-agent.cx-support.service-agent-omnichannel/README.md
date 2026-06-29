# Akka Sample: Service Agent

A single omnichannel service agent handles customer support cases arriving from WhatsApp, voice, web chat, and Facebook Messenger. The agent reads the case context, classifies the issue, and either resolves it directly or opens a CRM case and hands off to a human support representative — without following a rigid dialogue script.

Demonstrates the **single-agent** coordination pattern wired with four governance mechanisms: a PII sanitizer that runs before the agent ever sees customer message content, a `before-agent-response` guardrail that validates outbound replies before delivery, a `before-tool-call` guardrail that gates CRM write operations, and an application-level HITL escalation path that transfers control to a live agent.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- A valid credential for the CRM system you connect to. Supply it via the same key-sourcing options listed above (shell env var, env file, secrets-store URI, or one-time prompt) — Akka never writes the credential value to disk.

## Generate the system

```sh
cp -r ./single-agent.cx-support.service-agent-omnichannel  ~/my-projects/service-agent
cd ~/my-projects/service-agent
```

(Optional) Edit `SPEC.md` to narrow the channel set (for example, keep only web chat) or to point at a specific CRM integration by updating the tool-call list in Section 4.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ServiceAgent** — an AutonomousAgent that accepts an inbound customer message with channel context and returns a typed `AgentReply`.
- **CaseWorkflow** — orchestrates sanitize-wait → triage → handle per inbound message thread.
- **CaseEntity** — an EventSourcedEntity holding the per-case lifecycle.
- **MessageSanitizer** — a Consumer that subscribes to `MessageReceived` events, redacts PII, and emits `MessageSanitized` back to the entity.
- **ReplyGuardrail** — registered on `ServiceAgent` via the `before-agent-response` hook; validates every outbound reply before it leaves the agent loop.
- **CrmWriteGuardrail** — registered on `ServiceAgent` via the `before-tool-call` hook; gates CRM case write operations.
- **EscalationScorer** — a deterministic rule-based evaluator that fires after a reply is delivered, scoring whether the case was handled appropriately or should have escalated.
- **CaseView + CaseEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — restrict or extend the channel list; each channel maps to an `InboundChannel` enum value in the seeded data.
- `SPEC.md §5` — add domain-specific fields to `CaseContext` (e.g., `productLine`, `contractTier`, `languageCode`) to steer triage.
- `prompts/service-agent.md` — tailor the agent's persona, escalation thresholds, and tone policy for your deployment.
- `eval-matrix.yaml` — wire a real CRM by naming it in the `before-tool-call` guardrail's implementation paragraph; update the tool-call list in the CrmWriteGuardrail spec.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A customer sends a message on a supported channel → it is sanitized → the agent handles it → a reply is delivered and a case record appears in the UI.
2. The agent produces a reply that contains a prohibited phrase → the `before-agent-response` guardrail rejects it → the agent retries → a clean reply lands.
3. The agent attempts a CRM write with an invalid case priority → the `before-tool-call` guardrail blocks the write → the agent corrects the payload → the write succeeds.
4. A customer message flagged by the triage step as requiring a specialist triggers the HITL escalation path → the case is transferred to a human queue → the UI shows the case in `ESCALATED` state.

## License

Apache 2.0.
