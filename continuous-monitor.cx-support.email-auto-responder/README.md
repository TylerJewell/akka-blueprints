# Akka Sample: Email Auto Responder

A monitored support inbox where each inbound email is sanitized for PII, drafted into a contextual reply by an agent, optionally gated for human approval, and sent — all from one buildable folder.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- An AI key, sourced one of five ways at generation time (real key never written to disk): mock provider (no key), name an existing env var, point at an env file you maintain, a secrets-store URI, or type once into the session. Canonical env vars: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY`.
- Host software: none. This blueprint runs out of the box — the inbound mail source and the outbound send action are modeled inside the same service.

## Generate the system

```sh
cp -r ./continuous-monitor.cx-support.email-auto-responder  ~/my-projects/email-auto-responder
cd ~/my-projects/email-auto-responder
```

(Optional) Edit `SPEC.md` to rename the system or change the model provider.

In Claude Code, from inside the folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, then print the listening URL.

## What you'll get

- `InboxMonitor` (TimedAction) that continuously drips canned inbound emails into the system.
- `InboxQueue` (EventSourcedEntity) and `InboundEmailConsumer` (Consumer) that start one workflow per email.
- `ResponderWorkflow` (Workflow) with sanitize → draft → review-gate → send steps.
- `TriageAgent` (Agent) and `ResponderAgent` (AutonomousAgent) that classify and draft replies.
- `EmailEntity` (EventSourcedEntity) plus `EmailsView` (View) feeding a live SSE list.
- Two HttpEndpoints and a single self-contained 5-tab UI.

## Customise before generating

- System name and one-line pitch — SPEC.md Section 1.
- Model provider default — SPEC.md Section 11 identity block.
- Sensitivity policy (which emails need human approval) — `prompts/triage-agent.md` and the review-gate step in SPEC.md Section 4.

## What gets validated

- An inbound email appears in the UI in `DRAFTED` with PII redacted and a non-empty reply draft.
- A sensitive email pauses in `AWAITING_REVIEW`; approving it drives it to `SENT`.
- A non-sensitive email auto-sends without a human gate.
- An email left unreviewed past the threshold is escalated.

See `reference/user-journeys.md` for the full acceptance steps.

## License

Apache 2.0.
