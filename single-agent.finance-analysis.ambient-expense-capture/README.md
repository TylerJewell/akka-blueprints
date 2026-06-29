# Akka Sample: Ambient Expense Capture

A single expense-capture agent listens to voice or ambient text input, extracts line-item expense data, categorizes each item, and submits the categorized report to a downstream expense system. PII in receipts is redacted before the model sees the data; every tool call that would write to the expense system is gated by a before-tool-call guardrail.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a PII sanitizer that runs against raw receipt text before the agent processes it, and a before-tool-call guardrail that validates every expense submission before it reaches the external system.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The expense system integration is simulated in-process; no external ERP or expense platform account is required.

## Generate the system

```sh
cp -r ./single-agent.finance-analysis.ambient-expense-capture  ~/my-projects/ambient-expense
cd ~/my-projects/ambient-expense
```

(Optional) Edit `SPEC.md` to change the seeded expense categories (the JSONL file under `src/main/resources/sample-events/expense-categories.jsonl` after generation) or to add custom submission-validation rules.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ExpenseCaptureAgent** — an AutonomousAgent that accepts raw receipt or voice-transcript input as a task attachment, extracts and categorizes line items, and returns a typed `ExpenseReport`.
- **ExpenseWorkflow** — orchestrates sanitize-wait → capture → submit steps per submitted receipt.
- **ExpenseEntity** — an EventSourcedEntity holding the per-submission lifecycle.
- **ReceiptSanitizer** — a Consumer that subscribes to `ReceiptSubmitted` events, redacts PII into a `SanitizedReceipt`, and emits `ReceiptSanitized` back to the entity.
- **SubmissionGuardrail** — registered on the agent; validates every tool call that would write to the expense system before the tool executes.
- **ExpenseView + ExpenseEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded expense-category taxonomy for your own corporate chart of accounts.
- `SPEC.md §5` — add fields to `ExpenseLineItem` (e.g., `projectCode`, `costCenter`, `taxCode`) to match your ERP schema.
- `prompts/expense-capture-agent.md` — narrow the agent's extraction rules (e.g., restrict currencies, enforce per-diem limits, require receipt-image references for amounts above a threshold).
- `eval-matrix.yaml` — point the sanitizer at a production PII redaction library by naming it in the mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits raw receipt text → it is sanitized → captured → the categorized expense report appears in the UI.
2. A line item whose amount exceeds the configured policy limit triggers the before-tool-call guardrail, blocking submission until the line item is flagged for approval.
3. PII strings in the receipt (card numbers, employee names, addresses) never appear in the LLM call log; only the redacted form does.
4. A submission whose raw text is ambiguous produces a partial capture with flagged items rather than a refusal.

## License

Apache 2.0.
