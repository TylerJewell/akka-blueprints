# Architecture — Financial Model Builder

The four diagrams in PLAN.md describe the system from four complementary angles. This document explains each one and then covers the governance flow in detail.

---

## Component graph

The component graph shows how the eleven production components connect.

`ModelEndpoint` is the sole HTTP entry point for analyst-facing operations. It routes create/approve/reject commands to `FinancialModelEntity` and triggers the workflow. `AppEndpoint` serves the compiled Angular UI and static assets.

`FinancialModelPipelineWorkflow` owns the pipeline lifecycle. It calls `FinancialModelAgent` for the three task phases (extract, build, validate), pauses at `reviewStep` by observing entity state, then drives `FilingFidelityScorer` in `evalStep`.

`FinancialModelAgent` receives one task per invocation and selects tools from `ExtractTools`, `BuildTools`, or `ValidateTools` based on the current phase. Before each tool call, `ModelPhaseGuardrail` checks that the requested tool belongs to the active phase.

`FinancialModelEntity` is the sole state owner. It processes commands, emits events, and is the authoritative source of `ModelStatus`. `FinancialModelView` projects a read-optimised row from the entity event stream and serves it over SSE to the App UI tab.

---

## Interaction sequence

The sequence diagram traces journey J1 — the full happy path from ticker submission through analyst approval to evaluation.

Key observations:

- The workflow calls the agent three times (once per task phase), not once per pipeline run.
- `ModelPhaseGuardrail` intercepts before each tool call. An allowed result lets the tool proceed; a blocked result would cause the agent to report a phase violation, the workflow to emit `GuardrailRejected` via the entity, and the pipeline to fail.
- `reviewStep` is a pause point. The workflow does not call the agent here; it waits on entity state via `asyncEffect`. No thread is held during the wait.
- `FilingFidelityScorer` runs synchronously in `evalStep` without an LLM call. It reads the `FinancialModel` and `ValidationReport` from the workflow context.

---

## State machine

`FinancialModelEntity` has thirteen states. The forward path is linear:

```
CREATED → EXTRACTING → EXTRACTED → BUILDING → BUILT →
VALIDATING → VALIDATED → PENDING_REVIEW → APPROVED → EVALUATED
```

Two terminal failure branches exit from PENDING_REVIEW and from any active phase:

- `PENDING_REVIEW → REJECTED` on analyst rejection.
- `EXTRACTING / BUILDING / VALIDATING → FAILED` on unrecoverable pipeline error.

`GuardrailRejected` is an audit event only. It does not cause a state transition; the pipeline can continue after a guardrail block (the agent reports the block and the workflow decides whether to fail or retry).

---

## Entity model

The entity model diagram shows the read/write relationships between the four stateful components.

`FinancialModelEntity` emits all domain events. `FinancialModelView` reads those events and maintains one projected row per `modelId`. `FinancialModelPipelineWorkflow` both reads (to observe status changes) and writes (to submit commands). `FinancialModelAgent` is a pure function in this model — it receives inputs and returns typed records (`FilingData`, `FinancialModel`, `ValidationReport`); it does not hold state.

---

## Governance flow

### Phase-gate guardrail

`ModelPhaseGuardrail` operates at the before-tool-call hook. The agent's current phase is carried in the task context. The guardrail compares the tool's declared phase tag against the context. If they differ, the tool call is rejected and the agent receives a refusal message. The workflow then emits `GuardrailRejected` on the entity (audit-only) and fails the pipeline.

This prevents an agent instruction from skipping phases or calling a later-phase tool mid-EXTRACT — for example, calling `computeRatio` (a BUILD tool) before `parseFiling` has returned any line items.

### Analyst review gate

After `ModelValidated`, the entity transitions to `PENDING_REVIEW` and emits `ReviewRequested`. The workflow's `reviewStep` calls `asyncEffect` with an observer on the entity state. The workflow thread is released; the workflow resumes when the entity state changes to `APPROVED` or `REJECTED`.

The analyst interacts via `POST /api/models/{id}/approve` or `POST /api/models/{id}/reject`, optionally including an `analystNote`. The entity applies the command and emits `ModelApproved` or `ModelRejected`.

### Filing fidelity eval

`FilingFidelityScorer` runs in `evalStep` after observing `APPROVED`. It reads the full `FinancialModel` and `ValidationReport` from the workflow step context and executes four checks in sequence:

1. Every `ModelRow.derivation` value exists as a `LineItem.sourceRef` in the associated `FilingData`.
2. No `Assumption.assumedValue` deviates more than 20% from the median of matching `LineItem` values with the same `name` field.
3. `ValidationReport.confidence >= 60`.
4. No `ValidationFlag` in `ValidationReport.flags` has `severity = "CRITICAL"`.

Each passing check adds one point to the score. The scorer calls `entity.command(new ScoreModel(modelId, score, rationale))`, which emits `EvaluationScored` and transitions the entity to `EVALUATED`.
