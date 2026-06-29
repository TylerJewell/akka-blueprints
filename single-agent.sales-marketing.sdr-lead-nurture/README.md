# Akka Sample: SDR Agent

A single sales-development-rep agent nurtures inbound leads around the clock — answering product questions, handling common objections, and booking meetings directly onto an account executive's calendar. All outbound messages pass through a brand-safety guardrail before they are sent; every calendar and CRM write action passes through a second guardrail before the tool is called; lead contact details are sanitized out of the agent's prompt context before any LLM call.

Demonstrates the **single-agent** coordination pattern wired with three governance mechanisms: a PII sanitizer that strips contact details before the agent ever sees a lead record, a `before-agent-response` guardrail that validates every candidate reply for brand-safety, and a `before-tool-call` guardrail that checks write-action tool calls (calendar booking, CRM updates) before they are executed.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- A valid credential for the integrated CRM and calendar service. Supply it via the same five options as the model-provider key: existing shell env var, env file, secrets-store URI, or one-time session entry. Akka never writes the credential value to disk.

## Generate the system

```sh
cp -r ./single-agent.sales-marketing.sdr-lead-nurture  ~/my-projects/sdr-agent
cd ~/my-projects/sdr-agent
```

(Optional) Edit `SPEC.md` to point at your own CRM connector or to swap the seeded lead corpus for a live data feed.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **SdrAgent** — an AutonomousAgent that carries the entire lead-nurture conversation: answers questions, handles objections, qualifies intent, and books a meeting when the lead is ready.
- **LeadWorkflow** — orchestrates sanitize-wait → engage → close-or-handoff per lead session.
- **LeadEntity** — an EventSourcedEntity holding the per-lead lifecycle from inbound through qualified/booked/dismissed.
- **LeadSanitizer** — a Consumer that subscribes to `LeadReceived` events, strips PII from the contact record before it reaches the agent, and emits `LeadSanitized` back to the entity.
- **BookingGuardrail** — a `before-tool-call` guardrail that validates calendar-booking and CRM-mutation tool calls before execution.
- **ReplyGuardrail** — a `before-agent-response` guardrail that checks every candidate outbound message for brand-safety issues.
- **LeadView + LeadEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded lead corpus for a live webhook from your marketing automation platform.
- `SPEC.md §5` — extend `LeadRecord` with industry-specific qualification fields (e.g., `annualRevenue`, `techStack`, `budgetSignal`).
- `prompts/sdr-agent.md` — narrow the agent's positioning (product messaging, objection library, prohibited topics list).
- `eval-matrix.yaml` — wire real CRM and calendar integrations by naming the external client under the guardrail mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A lead comes in → it is sanitized → the agent engages → a meeting is booked and the CRM is updated.
2. The agent produces a reply that fails the brand-safety check on the first iteration → the `before-agent-response` guardrail rejects it → the agent retries → a compliant reply lands.
3. The agent attempts to book a calendar slot that is outside allowed hours → the `before-tool-call` guardrail blocks the call → the agent picks an allowed slot.
4. Lead contact details submitted with the inbound record never appear in the LLM call log; only the redacted form does.

## License

Apache 2.0.
