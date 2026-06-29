# Akka Sample: Statement Auditor

An auditor agent analyzes a financial statement section for internal consistency and source traceability; a reviewer agent scores each finding against a materiality rubric and validates working-paper citations; the two exchange findings and feedback until the reviewer signs off or the loop hits its retry ceiling. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance controls for the legal-compliance domain.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **none**. The blueprint runs out of the box; the inbound statement stream and the audit workspace are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.legal-compliance.statement-auditor  ~/my-projects/statement-auditor
cd ~/my-projects/statement-auditor
```

(Optional) Edit `SPEC.md` to change the materiality thresholds, the citation-validation rules, the max findings per statement, or the retry cap.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **AuditorAgent** — AutonomousAgent that analyzes a financial statement section, identifies consistency gaps, and produces a typed `FindingReport` (findings list with citations). On a revision call it incorporates reviewer feedback and re-examines the section.
- **ReviewerAgent** — AutonomousAgent that scores each finding against the materiality rubric, validates working-paper citations, and returns either `APPROVED` or `REVISE` with typed `ReviewNotes`.
- **AuditWorkflow** — Workflow that runs the analyze → citation-guardrail → review → revise loop up to a configurable retry ceiling, transitions the engagement to `SIGNED_OFF` on reviewer approval or to `UNRESOLVED` when the ceiling is hit.
- **EngagementEntity** — EventSourcedEntity that holds the engagement lifecycle, every finding iteration, every review, and the final outcome.
- **StatementQueue** — EventSourcedEntity that logs each submitted statement section for replay and audit.
- **EngagementsView** — read-side projection that the UI lists and streams via SSE.
- **StatementConsumer** — Consumer that starts a workflow per inbound submission.
- **StatementSimulator** — TimedAction that drips a sample statement section every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records per-cycle eval events (control E1).
- **AuditEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the statement sections the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Engagement` record fields (e.g., raise `maxAttempts`, add new finding categories).
- `prompts/auditor.md` — narrow the analysis to a different statement type (e.g., cash-flow statements, notes to accounts).
- `prompts/reviewer.md` — change the rubric (e.g., add a GAAP-compliance dimension).
- `eval-matrix.yaml` — tighten enforcement on the citation guardrail (currently blocking).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a statement section → engagement progresses `ANALYZING` → `REVIEWING` → `SIGNED_OFF` within the retry ceiling.
2. Force-fail rubric → engagement hits `UNRESOLVED` after the configured number of attempts; the entity preserves every finding iteration and every review for audit.
3. The citation guardrail blocks a finding report whose citations reference non-existent working papers, so the reviewer never sees unsupported findings.
4. Each completed cycle emits an `EvalRecorded` event that surfaces in the App UI's per-iteration timeline.

## License

Apache 2.0.
