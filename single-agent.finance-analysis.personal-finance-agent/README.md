# Akka Sample: Personal Finance Assistant

A single-agent personal-finance assistant accepts natural-language queries about a user's accounts, transactions, and spending, routes them through bank and finance tools, and returns structured answers. All tool calls that modify financial records pass through a before-tool-call guardrail before execution, and every transaction accessed by the agent is PII-sanitized before it reaches the model.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a PII sanitizer that strips account numbers, card PAN fragments, and full names before the model ever sees transaction data, and a before-tool-call guardrail that validates every write intent before the tool fires.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. All bank/finance tools are simulated in-process using seeded account and transaction data.

## Generate the system

```sh
cp -r ./single-agent.finance-analysis.personal-finance-agent  ~/my-projects/personal-finance-agent
cd ~/my-projects/personal-finance-agent
```

(Optional) Edit `SPEC.md` to adjust the seeded account data or the categories used by the expense-grouping tool.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **FinanceAssistantAgent** — an AutonomousAgent that accepts a user query plus context (account list, recent transactions) and calls finance tools to answer it. Returns a typed `AssistantResponse`.
- **QueryWorkflow** — orchestrates sanitize-wait → query → answer per submitted user query.
- **QueryEntity** — an EventSourcedEntity holding the per-query lifecycle.
- **TransactionSanitizer** — a Consumer that subscribes to `QuerySubmitted` events, redacts PII from transaction data, and emits `TransactionsSanitized` back to the entity.
- **FinanceTools** — the tool provider: `getBalance`, `listTransactions`, `groupByCategory`, `transferFunds`, `payBill`. Write tools (`transferFunds`, `payBill`) pass through the before-tool-call guardrail.
- **QueryView + QueryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded account and transaction data (the JSONL files under `src/main/resources/sample-events/` after generation).
- `SPEC.md §5` — extend `AssistantResponse` with additional fields (e.g., `suggestedActions`, `confidenceScore`).
- `prompts/finance-assistant.md` — narrow the agent's persona (restrict it to read-only queries, adjust the tool-call policy, or add spending-advice behaviour).
- `eval-matrix.yaml` — wire a real PII redactor (e.g., a presidio-style library) by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a spending-summary query → transactions are sanitized → the agent answers with grouped categories and totals visible in the UI.
2. The agent attempts a `transferFunds` call → the before-tool-call guardrail validates the intent → the transfer proceeds and the confirmation appears in the answer.
3. The agent attempts a `transferFunds` call with an amount that exceeds a seeded limit → the guardrail blocks it → the agent returns an error explanation without executing the transfer.
4. Account numbers and card PAN fragments in the transaction data never appear in the LLM call log; only tokenised forms do.

## License

Apache 2.0.
