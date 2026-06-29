# Akka Sample: Financial Advisor Agent

A single financial-advisor agent answers client investment questions by calling portfolio and market data tools, then returns a structured advisory response — a recommendation, risk assessment, and a per-holding breakdown — filtered through sector-level guardrails before any answer reaches the client.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a sector sanitizer that strips regulated financial identifiers before the agent receives client data, and a `before-agent-response` guardrail that blocks any response containing an unauthorized investment recommendation or an out-of-scope asset class.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — portfolio data and market prices are seeded in-process; the agent's tool calls resolve against that in-memory store.

## Generate the system

```sh
cp -r ./single-agent.finance-analysis.financial-advisor-agent  ~/my-projects/financial-advisor-agent
cd ~/my-projects/financial-advisor-agent
```

(Optional) Edit `SPEC.md` to narrow the instruction set — for example, restrict the agent to retirement accounts only, or add a second asset class (fixed income) to the allowed sectors.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **FinancialAdvisorAgent** — an AutonomousAgent that accepts a client advisory request (portfolio snapshot + question) as a task, calls portfolio and market data tools, and returns a typed `AdvisoryResponse`.
- **AdvisoryWorkflow** — orchestrates sanitize-wait → advise → audit steps per submitted request.
- **AdvisoryEntity** — an EventSourcedEntity holding the per-request lifecycle.
- **ClientDataSanitizer** — a Consumer that subscribes to `AdvisoryRequested` events, strips regulated financial identifiers, and emits `ClientDataSanitized` back to the entity.
- **AdvisoryView + AdvisoryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded client profiles and questions for your own (the JSONL file under `src/main/resources/sample-events/seed-clients.jsonl` after generation).
- `SPEC.md §5` — extend `AdvisoryResponse` with firm-specific fields (e.g., `regulatoryDisclosure`, `suitabilityScore`, `fiduciaryNotes`).
- `prompts/financial-advisor.md` — narrow the agent's role: a wealth-management deployer would add suitability rules; a robo-advisory deployer would constrain to passive index strategies only.
- `eval-matrix.yaml` — add regulation anchors (e.g., SEC Reg BI, MiFID II Article 25) once the deployer has confirmed jurisdictional scope.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A client question + portfolio is submitted → data is sanitized → the agent answers → the advisory appears in the UI.
2. The agent's first attempt on a request cites an unauthorized asset class → the `before-agent-response` guardrail rejects it → the agent retries → a compliant response lands.
3. Every advisory response carries a risk rating visible on the same UI card.
4. Account numbers and tax identifiers submitted with the client profile never appear in the LLM call log; only redacted tokens do.

## License

Apache 2.0.
