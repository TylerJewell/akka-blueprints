# Akka Sample: Bank Support Agent

A single banking support agent answers customer enquiries by looking up account data through a tool call and returning a structured `SupportResponse` — including a `riskScore`, an `answer` text, and a `blockCard` flag that signals whether card-blocking is recommended.

Demonstrates the **single-agent** coordination pattern wired with three governance mechanisms: a `before-tool-call` guardrail that vetoes card-blocking calls until pre-conditions are verified, a PII sanitizer that strips account numbers and personal identifiers from the response log, and a `before-agent-response` guardrail that validates the structured response before it reaches the customer channel.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The agent's account-lookup tool is simulated in-process; no external banking system is required.

## Generate the system

```sh
cp -r ./single-agent.cx-support.bank-support-agent  ~/my-projects/bank-support-agent
cd ~/my-projects/bank-support-agent
```

(Optional) Edit `SPEC.md` to point at a different account fixture set (e.g., switch from retail checking/savings to credit-card or mortgage accounts).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **BankSupportAgent** — an AutonomousAgent that receives a customer enquiry plus an account context, calls the `AccountLookupTool` to fetch live account data, and returns a typed `SupportResponse`.
- **EnquiryWorkflow** — orchestrates receive → sanitize-log → respond → record per submitted enquiry.
- **EnquiryEntity** — an EventSourcedEntity holding the per-enquiry lifecycle.
- **ResponseSanitizer** — a Consumer that subscribes to `ResponseRecorded` events and writes a PII-redacted copy to the log, so account numbers and names are never stored in the audit trail.
- **EnquiryView + EnquiryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded account fixture set for your own (the JSONL file under `src/main/resources/sample-events/accounts.jsonl` after generation).
- `SPEC.md §5` — extend `SupportResponse` with institution-specific fields (e.g., `escalationQueue`, `regulatoryFlag`, `productCode`).
- `prompts/bank-support-agent.md` — narrow the agent's scope to a specific product line (mortgage enquiries vs. card disputes vs. current-account queries).
- `eval-matrix.yaml` — replace the in-process card-blocking guardrail with a call to your institution's own authorization service.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A customer submits an enquiry → the agent looks up the account → a `SupportResponse` appears in the UI with a risk score and answer within 30 s.
2. The agent attempts to call `AccountLookupTool` with a `blockCard=true` parameter on a low-risk enquiry → the `before-tool-call` guardrail vetoes it → the call is blocked; the agent must re-plan.
3. The agent returns a `SupportResponse` with a risk score outside the valid range → the `before-agent-response` guardrail rejects it → the agent retries → a well-formed response reaches the UI.
4. A response containing the customer's account number and full name is recorded in the log → only the redacted form appears in `ResponseSanitizer`'s output → the raw identifiers are on the entity only.

## License

Apache 2.0.
