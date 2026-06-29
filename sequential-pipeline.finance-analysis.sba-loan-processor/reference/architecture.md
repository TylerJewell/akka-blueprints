# Architecture — sba-loan-processor

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs four tasks in sequence, wrapped by two sanitizers, one guardrail, and a durable human-in-the-loop pause. `LoanEndpoint` accepts a `{applicantRef, loanAmount, loanPurpose}` POST, writes `ApplicationSubmitted` onto `LoanApplicationEntity`, and starts `LoanPipelineWorkflow` keyed by `"pipeline-" + applicationId`. The workflow's first step (`intakeStep`) emits `IntakeStarted`, sanitizes the applicant record through `PiiSanitizer` to strip identifiers, then calls `LoanUnderwritingAgent` with `TaskDef.taskType(INTAKE_APPLICATION)` and a `phase = INTAKE` metadata tag. The agent invokes `IntakeTools.lookupCreditScore` and `IntakeTools.fetchBusinessProfile`; every tool call passes through `PhaseGuardrail` first.

Once the agent returns a `CreditProfile`, the workflow writes `IntakeCompleted` and advances to `underwriteStep`. Here `ProtectedAttributeSanitizer` masks the protected-attribute fields before the UNDERWRITE task context is assembled. The same sanitizer fires again before the DECISION task. After `DecisionMade` lands, `evalStep` runs `FairLendingScorer` over the `(LoanDecision, UnderwritingAnalysis, CreditProfile)` triple — no LLM call — and writes `FairnessEvaluated`. Then `hitlStep` emits `OfficerReviewRequested`, sets the entity to `PENDING_REVIEW`, and the workflow suspends. It resumes only when a loan officer submits a review via `POST /api/loans/{id}/review`. On approval, `reportStep` runs the GENERATE_REPORT task and writes `MemoGenerated`. `LoanApplicationView` projects every event into a read-model row; `LoanEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `FairLendingScorer` is deterministic and rule-based — that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Four properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `intakeStep` and `underwriteStep`, the workflow writes `IntakeCompleted{creditProfile}` onto the entity. The next step reads `creditProfile` from the entity to build the UNDERWRITE task's instruction context. The agent never sees intake-phase context inside the underwrite task's conversation; the typed handoff is the only path information travels.
2. Sanitizers fire before context assembly, not before tool calls. `PiiSanitizer` and `ProtectedAttributeSanitizer` are `before-call` hooks on the agent — they shape the instruction context sent to the model. `PhaseGuardrail` is a `before-tool-call` hook — it intercepts individual tool invocations. These are independent controls.
3. The HITL pause is durable. `hitlStep` uses Akka Workflow's native pause mechanism — the workflow state persists across restarts. The officer queue is not a soft "please check this" — the workflow cannot advance without an explicit officer action on `LoanEndpoint`.
4. The fairness score arrives before the officer's review, not after. The officer sees the score on the same UI card as the decision, so it informs their review rather than being a post-hoc report.

## State machine

Eleven states. The interesting paths:

- The happy path walks `SUBMITTED → INTAKE_IN_PROGRESS → INTAKE_COMPLETE → UNDERWRITING → UNDERWRITING_COMPLETE → DECISION_IN_PROGRESS → DECISION_MADE → PENDING_REVIEW → OFFICER_APPROVED`.
- The denial path branches at `PENDING_REVIEW → OFFICER_DENIED` (no memo generated).
- Three failure transitions land in `FAILED`: an agent error during INTAKE, UNDERWRITING, or DECISION. A failed application's partial state is preserved on the entity.
- `GuardrailRejected` and `FairnessEvaluated` are side-events that do not transition the primary status; they are recorded for audit and UI display.

There is no automated terminal state. Every application that reaches `DECISION_MADE` must pass through `PENDING_REVIEW` with a human actor. `OFFICER_APPROVED` or `OFFICER_DENIED` are the only valid terminal states for a completed application.

## Entity model

`LoanApplicationEntity` is the source of truth. It emits fourteen event types — four lifecycle starts, four lifecycle completions (including the memo), the fairness evaluation, the officer review request, the two officer terminal events, the guardrail audit, and the failure. `LoanApplicationView` projects every event into a row used by the UI. `LoanPipelineWorkflow` both reads and writes on the entity throughout the pipeline. The relationship between `LoanUnderwritingAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any application that lands in the entity log, the record passed through:

1. **PII sanitizer** — SSN, full name, DOB, and address stripped before the agent sees the application. Fires on every task.
2. **Protected-attribute sanitizer** — race, ethnicity, sex, age, marital status, and national origin masked before UNDERWRITE and DECISION tasks. The agent cannot reference what it cannot see.
3. **Phase-gate guardrail** — every tool call is filtered. A DECISION-phase tool called during INTAKE is rejected before the tool body runs; a `GuardrailRejected` event records the violation.
4. **LoanUnderwritingAgent (4 task runs)** — four model calls, four structured outputs. Each typed result is the dependency handoff to the next phase.
5. **Fair-lending on-decision eval** — every `LoanDecision` gets a 1–5 fairness signal score covering denial-rate pattern, rationale completeness, proxy-language absence, and DSCR correctness.
6. **Human-in-the-loop gate** — no decision takes effect without an officer's explicit approve or deny action on a durable workflow pause step.

Each control is independent. The sanitizers do not check phase order; the guardrail does not check proxy language; the evaluator does not check PII; the HITL gate does not score fairness. Removing any one opens an explicit gap the others do not cover.
