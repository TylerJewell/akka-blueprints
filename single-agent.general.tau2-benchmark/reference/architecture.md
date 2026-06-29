# Architecture — tau2-benchmark-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `BenchmarkEndpoint` accepts a run submission, writes a `RunSubmitted` event onto `BenchmarkRunEntity`, and immediately starts a `BenchmarkRunWorkflow` instance. The workflow's `executeStep` calls `BenchmarkAgent` — the single AutonomousAgent — with the task description as `TaskDef.instructions(...)` and the full `BenchmarkTask` JSON as a `TaskDef.attachment(...)`. Once the agent returns a `TaskResult`, the workflow writes `ResultRecorded` to the entity and calls `TaskScorer` in-process inside `scoreStep`. The score lands as `RunScored`. `BenchmarkView` projects every entity event into a read-model row; `BenchmarkEndpoint` serves the read model to the UI over REST, SSE, and an aggregate endpoint.

There is no second agent. `TaskScorer` is a deterministic rule-based class — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct moments where execution waits:

1. The `executeStep` timeout window — 60 s — during which `BenchmarkAgent` works through the task's steps and produces its `TaskResult`. LLM latency for multi-step reasoning can approach this limit.
2. The `scoreStep` — synchronous and in-process. Completes in milliseconds; no external call.

The aggregate panel in the UI is computed on every `GET /api/runs/aggregate` call from the full run list, so it reflects the latest SCORED run immediately after the SSE event arrives.

## State machine

Five states. Key paths:

- Happy path: `SUBMITTED → EXECUTING → RESULT_RECORDED → SCORED`.
- Two failure transitions land in `FAILED`: a workflow-start error during `SUBMITTED`, and an agent timeout or error during `EXECUTING`. A `FAILED` run's `request` is preserved on the entity — the UI shows the task definition even if no result landed.
- There is no `APPROVED` or `REVIEWED` state. Benchmark results are infrastructure outputs; humans inspect aggregates, not individual runs. The system deliberately stops at `SCORED`.

## Entity model

`BenchmarkRunEntity` is the source of truth. It emits five event types. `BenchmarkView` projects every event into a row used by the UI. `BenchmarkRunWorkflow` both reads (`getRun`) and writes (`markExecuting`, `recordResult`, `recordScore`, `fail`) on the entity. `BenchmarkAgent` returns a `TaskResult`; the workflow writes it to the entity via `recordResult`.

## Governance flow

Every result that lands in the entity log passed through:

1. **BenchmarkAgent** — one model call, one structured output.
2. **On-decision evaluator (`TaskScorer`)** — immediately after `ResultRecorded`, every result is scored on four completeness criteria. The same result always scores the same; no LLM involved in scoring.

Unlike the legal-compliance pattern, this baseline does not wire a pre-response guardrail. The tau2 benchmark surface does not require structural output validation before the result lands — the scorer's job is performance monitoring, not input sanitization or format enforcement. Deployers who need stricter output validation can add a `before-agent-response` guardrail by following the docreview blueprint as a reference.
