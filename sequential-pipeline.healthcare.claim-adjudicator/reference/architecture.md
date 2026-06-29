# Architecture — claim-adjudication-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `ClaimEndpoint` accepts a claim POST, writes `ClaimCreated` onto `ClaimEntity`, and starts `ClaimAdjudicationWorkflow` keyed by `"adj-" + claimId`. The workflow's first step (`validateStep`) emits `ValidationStarted`, then calls `AdjudicationAgent` with `TaskDef.taskType(VALIDATE_CLAIM)` and a `phase = VALIDATE` metadata tag. The agent invokes `ValidateTools.checkEligibility` and `ValidateTools.verifyProcedureCodes`; every call passes through `PhiSanitizerGuardrail` first. Once the agent returns a `ValidationResult`, the workflow writes `ClaimValidated` onto the entity and advances to `evaluateStep` — same pattern, the EVALUATE task carries `phase = EVALUATE`. Then `adjudicateStep` runs with `phase = ADJUDICATE`.

After `DecisionReached` lands, the workflow branches: if `outcome == DENIED`, it enters `humanReviewStep`, emitting `HumanReviewRequested` and blocking until a reviewer posts `approveDenial` or `overrideToApprove` to `ClaimEndpoint`. If `outcome == APPROVED`, the workflow proceeds directly to `evalStep`. In either branch, `evalStep` runs `AdjudicationScorer` over the recorded triple — no LLM call — and writes `EvaluationScored`. `ClaimView` projects every event into a read-model row; `ClaimEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `AdjudicationScorer` is a deterministic rule-based scorer; `PhiSanitizerGuardrail` is a deterministic sanitiser with no LLM call. That is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the approval path (J1). Three properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `validateStep` and `evaluateStep`, the workflow writes `ClaimValidated` onto the entity. The next step reads `validation` from the entity to build the EVALUATE task's instruction context. The agent never sees validate-phase context inside the evaluate task's conversation; the typed handoff is the only path information travels.
2. Every tool call is filtered through `PhiSanitizerGuardrail`. The sanitiser runs before the tool body executes, on every invocation. PHI fields are replaced with masked tokens; a `PhiRedacted` event is recorded for each field. After sanitisation, the same guardrail checks phase order and rejects out-of-phase calls.
3. The human-review hold runs outside the agent. When `adjudicateStep` returns a denial, the workflow emits `HumanReviewRequested` and blocks. The agent is not involved in the review step — it has already returned its `AdjudicationDecision`. The reviewer acts through a separate REST call.

Agent task calls are bounded by per-step timeouts (90 s on validate / evaluate / adjudicate). `humanReviewStep` is bounded by a configurable timeout (72 h production, 10 s dev). `evalStep` is synchronous and finishes in milliseconds.

## State machine

Thirteen states. The paths of interest:

- The approval happy path walks `CREATED → VALIDATING → VALIDATED → EVALUATING → EVALUATED_COVERAGE → DECIDING → DECIDED → ADJUDICATION_EVALUATED`.
- The denial path diverges at `DECIDED → PENDING_REVIEW`, then either `DENIED → ADJUDICATION_EVALUATED` (after `DenialApproved`) or `APPROVED → ADJUDICATION_EVALUATED` (after `DenialOverridden`).
- The escalation path: `PENDING_REVIEW → ESCALATED` when the review timeout fires. `ESCALATED` is terminal.
- Three failure paths land in `FAILED`: an agent error or PHI sanitisation error during `VALIDATING`, `EVALUATING`, or `DECIDING`.
- `PhiRedacted` and `GuardrailRejected` are side-events recorded for audit; they do not transition status.

There is no `PUBLISHED` or `NOTIFIED` state. The blueprint deliberately stops at `ADJUDICATION_EVALUATED`; the payer's downstream communication system is out of scope.

## Entity model

`ClaimEntity` is the source of truth. It emits fifteen event types — three lifecycle starts, three lifecycle completions, the evaluation, the human-review lifecycle (requested / approved / overridden / escalated), the PHI audit trail, the guardrail audit, the failure, and the initial creation. `ClaimView` projects every event into a row used by the UI. `ClaimAdjudicationWorkflow` both reads (`getClaim`) and writes extensively on the entity. The relationship between `AdjudicationAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any claim that lands in the entity log, the claim passed through:

1. **PHI sanitiser** — every tool call is sanitised. No raw PHI enters a tool body, a tool log, or any downstream store. The `PhiRedacted` event stream gives the compliance team a queryable, claim-scoped redaction log.
2. **AdjudicationAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **Human-review hold** — any denial is intercepted before finalisation. The reviewer sees the full decision rationale, coverage evaluation, and denial code before confirming or overriding.
4. **On-decision evaluator** — every finalised decision gets a 1–5 score. Rule citation count, code provenance, outcome consistency, and evidence completeness are each worth one point on a base of 1.

Each layer is independent. The sanitiser does not check rule citations; the evaluator does not check PHI exposure; the human-review hold does not score evidence completeness. Removing one layer opens an explicit gap the others do not silently cover.
