# Akka Sample: Grid Impact Study Runner

A single `StudyAgent` walks an interconnection request through three task phases — **COLLECT → ANALYZE → REPORT** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a phase-specific tool set. An engineer submits a request and receives a structured `StudyReport`. An approval gate holds the report before issue.

Demonstrates the **sequential-pipeline** coordination pattern wired with two governance mechanisms: a human-in-the-loop approval gate that blocks report issuance until a qualified engineer confirms the findings, and an `on-decision-eval` evaluator that scores every emitted study report against simulator ground truth for accuracy and completeness.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every load-flow, contingency, and drafting tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.energy-utilities.impact-study-runner  ~/my-projects/impact-study-runner
cd ~/my-projects/impact-study-runner
```

(Optional) Edit `SPEC.md` to point at a different interconnection request corpus, a different model provider, or a richer set of simulation tools.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **StudyAgent** — one AutonomousAgent declaring three Task constants (`RUN_LOAD_FLOW`, `RUN_CONTINGENCY`, `DRAFT_REPORT`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **StudyPipelineWorkflow** — runs `loadFlowStep → contingencyStep → draftStep → evalStep → approvalStep`. Each step calls `runSingleTask` and writes the typed result back onto `StudyEntity` before the next step starts. The final `approvalStep` pauses the workflow until an engineer calls the approval endpoint.
- **StudyEntity** — an EventSourcedEntity holding the per-study lifecycle (`LoadFlowComputed`, `ContingencyAnalyzed`, `ReportDrafted`, `EvaluationScored`, `StudyApproved`, `StudyRejected`).
- **LoadFlowTools / ContingencyTools / DraftTools** — three function-tool classes registered on the agent, one per phase. A `before-tool-call` guardrail does not apply here; phase isolation is enforced by the workflow's per-step task scoping.
- **SimulatorScorer** — deterministic, rule-based on-decision evaluator that runs immediately after `ReportDrafted` and scores the report against simulator ground-truth data.
- **StudyView + StudyEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded request set under `src/main/resources/sample-events/requests.jsonl` to fit your demo bus topology.
- `SPEC.md §4` and `prompts/study-agent.md` — narrow the agent's role (e.g., constrain it to distribution-level studies, to offshore wind interconnection, to battery storage projects) by tightening the system prompt and renaming the typed records.
- `SPEC.md §5` — extend the typed outputs (`LoadFlowResult`, `ContingencyResult`, `StudyReport`) with additional grid parameters. The workflow task chaining does not need editing — it carries typed results across steps regardless of field shape.
- `eval-matrix.yaml` — wire a real simulator comparison (replace the deterministic stub with a call to your grid simulator's REST API) by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits an interconnection request → `LOAD_FLOW` runs → `CONTINGENCY` runs → `DRAFT` runs → a typed `StudyReport` lands awaiting engineer approval within ~60 s. Every transition is visible in real time.
2. An engineer approves the report via the approval endpoint → the study transitions to `APPROVED` and the report becomes retrievable as a final issued document.
3. Every `StudyReport` emitted has an on-decision eval score visible on the same UI card; reports that violate N-1 contingency thresholds or miss required bus-voltage checks receive a score ≤ 2 and are flagged before the approval gate.
4. Each task receives only its own typed inputs; the LOAD_FLOW task does not see the drafting instructions, and the DRAFT task does not see raw bus-voltage tables — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
