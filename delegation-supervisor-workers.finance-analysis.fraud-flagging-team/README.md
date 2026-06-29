# Akka Sample: Fraud Flagging Team

A supervisor agent delegates each customer transaction to fraud-scoring, compliance, and risk-assessment workers, synthesizes their findings into a fraud verdict, and pauses for a human analyst to confirm before any account action runs.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- An AI provider key. One of `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` in your shell. If none is set, `/akka:specify` offers a mock-LLM path that needs no key.
- Host software: none. This blueprint runs out of the box — the transaction feed and the account-action target are modeled inside the service.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.finance-analysis.fraud-flagging-team  ~/my-projects/fraud-flagging-team
cd ~/my-projects/fraud-flagging-team
```

(Optional) Edit `SPEC.md` — system name, model provider, transaction sample data.

In Claude Code, from inside the folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- One supervisor agent (AutonomousAgent) that plans delegation and synthesizes a fraud verdict.
- Three worker agents (Agent): fraud scoring, compliance check, customer risk assessment.
- A workflow that fans out to the workers, synthesizes, then pauses for analyst confirmation.
- An event-sourced case entity plus a read-model view and SSE stream.
- A transaction queue entity, a consumer that opens a case per transaction, and two timed actions (a feed simulator and a stuck-case monitor).
- Two HTTP endpoints (API + static UI) and a single self-contained UI with five tabs.

## Customise before generating

- `SPEC.md` Section 1 — system name and pitch.
- `SPEC.md` Section 11 — model provider and HTTP port.
- `reference/data-model.md` — transaction and case fields.
- `prompts/` — the four agent system prompts.

## What gets validated

- A submitted transaction opens a case, runs the three workers, and reaches a `FLAGGED` or `CLEARED` verdict.
- A `FLAGGED` case pauses for analyst review; confirming it runs the gated account action and reaches `ACTIONED`.
- Dismissing a flagged case reaches `DISMISSED` with no account action.
- A flagged case left untouched past the review window auto-escalates.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
