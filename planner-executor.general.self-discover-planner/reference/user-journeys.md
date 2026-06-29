# User journeys — self-discover-planner

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path: full plan-evaluate-execute cycle

**Preconditions:** Service running on the declared port; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:9699/`. App UI tab is visible.
2. In the Task prompt field, type "What are the tradeoffs between consistency and availability in distributed databases?" Click Submit.
3. A new solve card appears with status PLANNING.

**Expected:**
- Within 5 s, status transitions to EVALUATING via SSE.
- The expanded card shows a `ReasoningPlan` with 3–7 steps before execution starts.
- A `PlanEval` entry appears with `verdict = PASS` and a `score >= 0.65`.
- Status transitions to EXECUTING after the plan is accepted.
- Within ~3 minutes, status transitions to COMPLETED.
- The expanded view shows:
  - An accepted `ReasoningPlan` with a non-empty `stepOrder`.
  - An `ExecutionLog` with one entry per plan step. Each entry has a non-empty `result.output`.
  - A `TaskAnswer` with a 60–120 word summary and 3–5 evidence bullets citing module kinds and step indices.
- `planRevisionCount = 0`.

## J2 — Plan rejected and revised before execution

**Preconditions:** As J1, with a mock evaluator response seeded to return `FAIL` on the first plan.

**Steps:**
1. Submit "Evaluate the security tradeoffs of symmetric versus asymmetric encryption."
2. Observe the first `SolvePlanned` event fires, followed by `PlanRejected` (not `PlanAccepted`).
3. Observe a second `SolvePlanned` event fires (revision), followed by `PlanAccepted`.
4. Observe the first `ModuleExecuted` event only fires after `PlanAccepted`.

**Expected:**
- The expanded card shows `planRevisionCount = 1`.
- The plan eval section shows the rejected first eval (score < 0.65, `feedback` non-empty) and the passing second eval (score >= 0.65).
- Execution proceeds normally after the revised plan passes.
- The task completes with a `TaskAnswer` drawn from the revised plan's module steps.
- The workflow never enters the execution loop until `PlanAccepted` has been emitted — verified by the SSE event order: `SolvePlanned` → `PlanRejected` → `SolvePlanned` → `PlanAccepted` → `ModuleExecuted` (first).

## J3 — Eval gate ordering is provable from the event stream

**Preconditions:** As J1.

**Steps:**
1. Submit any task.
2. Subscribe to `GET /api/solves/sse` (or observe the live event sequence in the UI).
3. Record the timestamps of `SolvePlanned`, `PlanAccepted`, and the first `ModuleExecuted` event.

**Expected:**
- `SolvePlanned.timestamp < PlanAccepted.timestamp < first ModuleExecuted.timestamp`.
- No `ModuleExecuted` event appears before `PlanAccepted` for any solve.
- `GET /api/solves/{id}` returns a solve whose `planEval.verdict = "PASS"` whenever `executionLog.steps` is non-empty, and whose `planEval` is absent or null when the solve is still in `PLANNING` state.

## J4 — Output scrubber redacts a secret-shaped string in module output

**Preconditions:** As J1, with the `executor.json` mock entry for the ANALYSE module seeded to include the literal `AKIAIOSFODNN7EXAMPLE` in its output.

**Steps:**
1. Submit "Analyse the configuration files for legacy AWS credentials."
2. Observe the solve reaches COMPLETED.

**Expected:**
- The `ExecutionLog` entry for the module step whose mock output contained `AKIAIOSFODNN7EXAMPLE` shows `result.output` containing `[REDACTED:aws-access-key]` — not the literal key.
- `GET /api/solves/{id}` confirms the raw key is absent from every field: `executionLog`, `answer.summary`, and `answer.evidence`.
- The SSE `solve-update` payloads for `ModuleExecuted` do not contain the literal key.
- The UI's expanded execution log renders the redacted span in italics with a tooltip showing `aws-access-key`.
- The `SynthesiserAgent`'s input (the `ExecutionLog` passed at synthesis time) contains only the redacted form, so the `TaskAnswer.summary` cannot contain the raw key even in theory.
