# Akka Sample: LLM Auditor

A chatbot response passes through a critic agent that evaluates quality, compliance, and tone; a reviser agent rewrites flagged responses before they reach the end user. The two agents iterate inside a bounded workflow until the critic approves the revised output or the retry ceiling is hit. Demonstrates the **evaluator-optimizer** coordination pattern applied to governance risk — continuous post-generation auditing with a response guardrail.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; both the inbound response feed and the outbound audit surface are modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.governance-risk.llm-auditor  ~/my-projects/llm-auditor
cd ~/my-projects/llm-auditor
```

(Optional) Edit `SPEC.md` to change the audit rubric, the response quality thresholds, the per-session revision ceiling, or the severity scoring range.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CriticAgent** — AutonomousAgent that audits a chatbot response against a rubric (accuracy, tone, policy compliance, hallucination risk) and returns either `PASS` or `REVISE` with a typed `AuditFindings` payload.
- **ReviserAgent** — AutonomousAgent that rewrites a flagged response, addressing every finding in the audit before returning the revised text.
- **AuditWorkflow** — Workflow that runs the response → audit → revise loop up to a configurable ceiling, transitions the session record to `APPROVED` on critic pass or to `ESCALATED` when the ceiling is hit without a pass.
- **SessionEntity** — EventSourcedEntity that holds the audit lifecycle, every revision attempt, every set of findings, and the final disposition.
- **ResponseQueue** — EventSourcedEntity that logs each inbound chatbot response for replay and compliance record-keeping.
- **SessionsView** — read-side projection that the UI lists and streams via SSE.
- **ResponseIngestConsumer** — Consumer that starts a workflow per inbound response event.
- **ResponseSimulator** — TimedAction that drips a sample chatbot response every 60 s so the App UI is not empty.
- **AuditSampler** — TimedAction that records the per-cycle eval event each tick (control E1).
- **AuditEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample responses the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Session` record fields (e.g., raise `maxRevisions`, change the severity scoring range).
- `prompts/critic.md` — extend the rubric (e.g., add a regulatory citation requirement or a sentiment constraint).
- `prompts/reviser.md` — narrow the rewrite style (e.g., restrict vocabulary, add a citation format).
- `eval-matrix.yaml` — tighten enforcement on the response guardrail (currently blocking) or add a regulation anchor.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a chatbot response → session progresses `AUDITING` → `REVISING` → `APPROVED` within the revision ceiling.
2. Force the critic to always flag → session hits `ESCALATED` after the configured number of revisions; the entity preserves every revision and every findings set for audit.
3. The response guardrail blocks a response whose severity score exceeds the blocking threshold, sending it directly to `ESCALATED` without entering the revision loop.
4. Each completed cycle emits an `AuditRecorded` event that surfaces in the App UI's per-revision timeline.

## License

Apache 2.0.
