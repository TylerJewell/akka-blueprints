# Akka Sample: Multi-Modal Email Assistant

A single email-triage agent reads incoming emails — including attached files and inline images — classifies each message by urgency and category, drafts a reply, and presents the draft to the user for confirmation before any outbound send is attempted. The raw email body is sanitized of PII before the agent ever processes it, and a `before-tool-call` guardrail intercepts every send action for human approval.

Demonstrates the **single-agent** coordination pattern wired with three governance mechanisms: a PII sanitizer that strips identifiers from email content before the agent call, a `before-tool-call` guardrail that blocks outbound sends until the operator approves, and a human-in-the-loop confirmation step that gates every draft before delivery.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the email corpus lives in-process and outbound send is simulated by the mock tool.

## Generate the system

```sh
cp -r ./single-agent.ops-automation.email-triage-assistant  ~/my-projects/email-triage-assistant
cd ~/my-projects/email-triage-assistant
```

(Optional) Edit `SPEC.md` to adjust the triage categories (e.g., switch from the default support/billing/legal/general set to a domain-specific taxonomy), or update the urgency thresholds in the classification rules.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **EmailTriageAgent** — an AutonomousAgent that reads a sanitized email (body + attachment content), classifies it, and drafts a reply.
- **TriageWorkflow** — orchestrates sanitize-wait → triage → confirm → send steps per submitted email.
- **EmailEntity** — an EventSourcedEntity holding the per-email lifecycle.
- **EmailSanitizer** — a Consumer that subscribes to `EmailSubmitted` events, strips PII from the body and attachment text, and emits `EmailSanitized` back to the entity.
- **SendGuardrail** — the `before-tool-call` guard that intercepts every outbound-send tool invocation and blocks it until `ConfirmationEntity` records an approval.
- **ConfirmationEntity** — a lightweight EventSourcedEntity that records the user's approve/reject decision and notifies the workflow.
- **TriageView + TriageEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded email corpus for your own (the JSONL file under `src/main/resources/sample-events/seed-emails.jsonl` after generation).
- `SPEC.md §5` — extend `TriageResult` with domain-specific fields (e.g., `ticketId`, `escalationPath`, `responseDeadline`).
- `prompts/email-triage-agent.md` — narrow the agent's role (a legal-ops deployer would constrain it to regulatory-inquiry routing; a finance deployer to payment-dispute triage).
- `eval-matrix.yaml` — wire a real PII redactor (e.g., a presidio-style library) under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits an email with attachments → the body and attachment text are sanitized → the agent classifies and drafts a reply → the draft appears for confirmation in the UI.
2. The agent tries to call the outbound-send tool before the user has confirmed → the `before-tool-call` guardrail blocks the call → the user approves → the call proceeds and the entity transitions to `SENT`.
3. The user rejects a draft → the entity transitions to `REJECTED`; no outbound call is ever attempted.
4. PII strings in the submitted email body never appear in the LLM call; only the redacted form does, while the entity preserves the raw text for audit.

## License

Apache 2.0.
