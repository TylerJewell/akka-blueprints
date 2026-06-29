# Architecture — doc-linter

The system runs one agent over a single request, calling a deterministic tool to do the
mechanical work and using the model only to rank and summarize. The four diagrams below are
the same ones the Architecture tab renders; the source lives in `PLAN.md`.

## Component graph

One agent (`LinterAgent`) sits between the HTTP surface and the event-sourced record. The
agent owns the custom `MarkdownLintTool`. `LintEntity` records each run; `LintView` projects
runs for query and SSE. `RuleProfileEntity` holds learned per-rule weights that the agent
reads when ranking, and `FeedbackConsumer` updates those weights from recorded feedback.
`LintSimulator` keeps the run list populated by seeding sample paths on a timer.

See the `flowchart TB` in `PLAN.md` → Component graph.

## Interaction sequence

The primary journey: a POST to `/api/lint` creates a `LintEntity` in `REQUESTED`, the
endpoint calls `LinterAgent.lint`, the agent calls `MarkdownLintTool` (where the
before-tool-call guardrail confines the path), the tool returns findings, the agent returns
an ordered result and summary, and the endpoint records the result with a computed gate. The
UI sees the transition over SSE.

See the `sequenceDiagram` in `PLAN.md` → Interaction sequence.

## State machine

A run moves `REQUESTED → LINTED` on a successful lint, or `REQUESTED → BLOCKED` when the
guardrail rejects the path. From `LINTED`, recording feedback moves it to `FEEDBACK_APPLIED`,
which is re-entrant.

See the `stateDiagram-v2` in `PLAN.md` → State machine. The Architecture tab applies the
Lesson 24 CSS overrides so state names and transition labels stay legible.

## Entity model

`LintEntity` holds a `LintRun` with its findings and optional lifecycle fields;
`RuleProfileEntity` holds the weight map; `LintView` carries a projection used for query and
streaming.

See the `erDiagram` in `PLAN.md` → Entity model.
