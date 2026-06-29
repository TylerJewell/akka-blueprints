# Akka Sample: Data Warehouse Optimizer

An optimizer agent proposes query rewrites and schema changes for a data warehouse; an evaluator agent scores each proposal against cost, correctness, and safety rules; the two iterate until the evaluator approves or the loop reaches its attempt ceiling. A DBA approval gate blocks any DDL change from executing until a human confirms. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; the inbound request stream and the warehouse execution surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.dev-code.warehouse-optimizer  ~/my-projects/warehouse-optimizer
cd ~/my-projects/warehouse-optimizer
```

(Optional) Edit `SPEC.md` to change the optimization rubric, the DDL-safety rules, the attempt ceiling, or the DBA approval timeout.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **OptimizerAgent** — AutonomousAgent that proposes a rewritten query or schema change for a given optimization request, incorporating prior evaluator feedback on revisions.
- **EvaluatorAgent** — AutonomousAgent that scores a proposal against cost savings, correctness, safety, and DDL-risk criteria; returns either `APPROVE` or `REVISE` with a typed `EvaluationNotes` payload.
- **OptimizationWorkflow** — Workflow that runs the propose → guardrail → evaluate → revise loop up to a configurable attempt ceiling, transitions the request to `APPROVED` on evaluator approval or to `REJECTED_FINAL` when the ceiling is hit.
- **OptimizationRequestEntity** — EventSourcedEntity that holds the request lifecycle, every proposal, every evaluation, and the final outcome.
- **RequestQueue** — EventSourcedEntity that logs each submitted optimization request for replay and audit.
- **RequestsView** — read-side projection that the UI lists and streams via SSE.
- **RequestConsumer** — Consumer that starts a workflow per inbound submission.
- **RequestSimulator** — TimedAction that drips a sample optimization request every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-attempt eval event each cycle (control E1).
- **DBAApprovalGate** — HITL component that pauses the workflow on DDL proposals and waits for a DBA decision (control H1).
- **OptimizerEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customize before generating

- `SPEC.md §3` — change the requests the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `OptimizationRequest` record fields (e.g., raise `maxAttempts`, expand `ProposalKind` to add new rewrite types).
- `prompts/optimizer.md` — narrow the proposal style (e.g., restrict to index-only recommendations, exclude partition pruning).
- `prompts/evaluator.md` — change the rubric (e.g., add a query-plan-row-estimate threshold).
- `eval-matrix.yaml` — tighten the DDL guardrail enforcement or adjust the HITL timeout.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a query-rewrite request → request progresses `PROPOSING` → `EVALUATING` → `APPROVED` within the retry ceiling.
2. Force-fail the rubric → request hits `REJECTED_FINAL` after the configured number of attempts; the entity preserves every proposal and every evaluation for audit.
3. Submit a DDL request → the DDL guardrail fires before evaluation; DBA approval gate pauses the workflow and waits for human confirmation before continuing.
4. Each completed cycle emits an `EvalRecorded` event that surfaces in the App UI's per-attempt timeline.

## License

Apache 2.0.
