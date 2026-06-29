# Architecture — bank-support-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `EnquiryEndpoint` accepts a submission and writes an `EnquirySubmitted` event onto `EnquiryEntity`, then immediately starts an `EnquiryWorkflow` for that enquiryId. The workflow's `lookupStep` calls `AccountLookupTool` directly (in-process) to retrieve the `AccountSummary`, then writes `AccountLoaded` to the entity. The `respondStep` calls `BankSupportAgent` — the single AutonomousAgent — with the customer context as `TaskDef.instructions(...)`. The agent calls `AccountLookupTool` for account data; every `AccountLookupTool` invocation passes through `ToolCallGuardrail` first. Every candidate `SupportResponse` passes through `ResponseGuardrail` before leaving the agent loop. Once a response passes both guardrails, the workflow writes `ResponseRecorded` and runs `ResponseSanitizer` inside `sanitizeStep`. The sanitized log lands as `LogSanitized`. `EnquiryView` projects every entity event into a read-model row that shows only the sanitized response; `EnquiryEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `ResponseSanitizer` is a regex-based redaction pipeline — not an LLM call — keeping this blueprint's single-agent promise honest.

## Interaction sequence

The sequence traces the happy path (J1). Three distinct processing stages are visible:

1. **`lookupStep`** — the workflow calls `AccountLookupTool` synchronously in-process. Sub-second; bounded by the 10 s step timeout.
2. **`respondStep`** — the agent call, potentially multi-turn if guardrails trigger retries. The 60 s timeout accommodates LLM latency plus retry overhead.
3. **`sanitizeStep`** — the in-process PII redaction pipeline. Milliseconds; bounded by the 5 s step timeout.

The agent's `before-tool-call` guardrail fires inside the agent loop during `respondStep` — invisible in the top-level sequence but shown in the detailed agent-internal flow (step 10–14 of the sequence diagram).

## State machine

Six states. The notable paths:

- Happy path: `SUBMITTED → ACCOUNT_LOADED → RESPONDING → RESPONSE_RECORDED → LOGGED`.
- Two failure transitions land in `FAILED`: a lookup error during `ACCOUNT_LOADED`, and an agent error (or guardrail-exhaustion) during `RESPONDING`. A `FAILED` enquiry's partial data is preserved on the entity; the UI shows the last known state.
- There is no `RESOLVED` or `ACTIONED` state. The response is advisory; a human support agent or downstream authorization system acts on the `blockCard` flag. This blueprint deliberately stops at `LOGGED`.

## Entity model

`EnquiryEntity` is the source of truth. It emits six event types. `EnquiryView` projects every event into a row used by the UI — but the view holds only `sanitizedLog.redactedAnswer`, not `response.answer`. `EnquiryWorkflow` both reads (`getEnquiry`) and writes (`loadAccount`, `startResponding`, `recordResponse`, `attachSanitizedLog`, `fail`) on the entity. The relationship between `BankSupportAgent` and `SupportResponse` is "returns" — the agent's task result is the response record. `AccountLookupTool` is called by both the workflow (`lookupStep`) and the agent (`respondStep`); the diagram shows both paths.

## Defence-in-depth governance flow

For any response that reaches `LOGGED`, the enquiry passed through:

1. **before-tool-call guardrail (`ToolCallGuardrail`)** — card-block calls require risk threshold; all other lookups pass freely.
2. **`BankSupportAgent`** — one model call, one structured output.
3. **before-agent-response guardrail (`ResponseGuardrail`)** — bad parses, out-of-range risk scores, and inconsistent blockCard flags are caught before the response leaves the agent loop.
4. **PII sanitizer (`ResponseSanitizer`)** — account numbers, names, and contact details are stripped from the audit log before the view is updated.

Each step is independent. Removing one opens an explicit gap the others do not silently cover.
