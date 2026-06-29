# Akka Sample: Case Management Agent

A single support-agent receives inbound customer messages around the clock, creates and updates cases in a CRM-compatible store, and routes them to the right tier — all while enforcing two guardrails (one on every agent response, one before each CRM write) and stripping customer PII before the model ever sees it.

Demonstrates the **single-agent** coordination pattern wired with three governance mechanisms: a PII sanitizer that runs before the agent sees any customer input, a `before-agent-response` guardrail that validates the agent's proposed case action on every turn, and a `before-tool-call` guardrail that checks each CRM write for mandatory fields and policy constraints before execution.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the CRM store lives in-process and the agent's tool calls operate against an in-memory case repository.

## Generate the system

```sh
cp -r ./single-agent.cx-support.case-management-agent  ~/my-projects/case-management-agent
cd ~/my-projects/case-management-agent
```

(Optional) Edit `SPEC.md` to change the seeded case categories (e.g., switch from billing/technical/account-access to industry-specific categories), or adjust the routing-tier thresholds in Section 3.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **SupportAgent** — an AutonomousAgent that reads the sanitized customer message, classifies the case, decides on an action (create/update/escalate/close), and invokes the appropriate CRM tool.
- **CaseWorkflow** — orchestrates sanitize-wait → agent-turn → tool-execution → eval per inbound message.
- **CaseEntity** — an EventSourcedEntity holding the per-case lifecycle from open to resolved.
- **MessageSanitizer** — a Consumer that subscribes to `MessageReceived` events, redacts PII into a `SanitizedMessage`, and emits `MessageSanitized` back to the entity.
- **CaseView + CaseEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded case categories and routing tiers for your own business taxonomy.
- `SPEC.md §5` — extend `CaseRecord` with industry-specific fields (e.g., `contractId`, `productSku`, `slaClass`).
- `prompts/support-agent.md` — narrow the agent's role (a telecom deployer would constrain it to network-fault triage; a SaaS deployer to tier-1 account operations).
- `eval-matrix.yaml` — wire a real PII redactor by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A customer message arrives → it is sanitized → a case is created by the agent → the case appears in the UI with the correct tier and status.
2. The agent proposes a case action whose priority field is missing — the `before-agent-response` guardrail rejects it — the agent retries — a well-formed action lands.
3. A tool call that would write a case with a disallowed escalation path is blocked by the `before-tool-call` guardrail before it reaches the CRM store.
4. PII strings in the inbound message never appear in the LLM call log; only the redacted form does.

## License

Apache 2.0.
