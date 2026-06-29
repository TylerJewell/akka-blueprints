# Architecture — hr-shortlister

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `ShortlistEndpoint` accepts a `{jobId, resumeText}` POST, writes `ResumeReceived` onto `ApplicationEntity`, and starts `ShortlistingWorkflow` keyed by `"shortlist-" + applicationId`. The workflow's first step (`parseStep`) emits `ParseStarted`, then calls `ShortlistAgent` with `TaskDef.taskType(PARSE_RESUME)` and a `phase = PARSE` metadata tag. The agent invokes `ParseTools.extractProfile` and `ParseTools.normalizeEducation` and returns a `Profile`. The workflow writes `ProfileParsed` onto the entity and advances to `scoreStep` — the `Profile` becomes the SCORE task's instruction context. Before any SCORE-phase tool body executes, `SpecialCategoryGuardrail` intercepts the call, sanitizes the profile (replacing protected-attribute values with `"[REDACTED]"`), and records each replacement as a `SpecialCategoryRedacted` event. After `scoreStep` returns a `CandidateScore` and `shortlistStep` returns a `ShortlistDecision`, the workflow emits `ApprovalRequested` and suspends. The workflow resumes only when the recruiter POSTs a decision through `ShortlistEndpoint`. After recruiter action, `erpNextStep` calls `ErpNextAdapter` and emits `WrittenToErpNext`.

Running in parallel with the per-application pipeline is `FairnessDriftWorkflow`, a scheduled workflow that fires daily. It reads the 30-day application window from `ApplicationView`, passes the rows to `FairnessScorer`, and writes the resulting `DriftReport` records to `DriftReportEntity`. `ShortlistEndpoint` serves the current drift reports at `GET /api/drift-reports`.

The graph has exactly one LLM-calling component. `FairnessScorer` and `SpecialCategoryGuardrail` are deterministic in-process logic — no LLM calls — which is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1): resume submitted, all three phases complete, recruiter approves, ERPNext stub called.

Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `parseStep` and `scoreStep`, the workflow writes `ProfileParsed` onto the entity. `scoreStep` reads `profile` from the entity to build the SCORE task's instruction context. The agent never sees raw resume text inside the score task's conversation; the typed `Profile` handoff is the only path information travels.
2. The sanitizer fires at the tool-call boundary, not at the task boundary. The `SpecialCategoryGuardrail` intercepts each `evaluateCriteria` and `computeOverallScore` call individually, replacing protected-attribute field values just before the tool body runs. This means the sanitizer coverage is unconditional — it fires even if the agent constructs the `Profile` argument inline rather than from the entity state.

The `approvalStep` suspension is deliberate: `effects().pause()` is the Akka mechanism for human-paced gates. The workflow resumes on an explicit HTTP call from the recruiter, not on a timer.

## State machine

Eleven states. The interesting paths:

- The happy path walks `RECEIVED → PARSING → PARSED → SCORING → SCORED → DECIDING → DECIDED → AWAITING_APPROVAL → APPROVED → WRITTEN`.
- Three failure transitions land in `FAILED`: agent error during `PARSING`, `SCORING`, or `DECIDING`. A failed application's prior data is preserved on the entity.
- `SpecialCategoryRedacted` is a side-event (zero status transitions). It records which fields were sanitized, when, and for which application. The recruiter UI surfaces these events in the redaction-log strip.
- `RecruiterOverridden` transitions to `APPROVED` on the same arc as `RecruiterApproved` — the status target is the same, but the payload differs. The ERPNext write uses the recruiter's overriding decision, not the agent's original output.

There is no `REJECTED` terminal state. A `REJECT` decision by the agent or recruiter still reaches `WRITTEN` — the rejection is the ERPNext record content, not a workflow abort.

## Entity model

`ApplicationEntity` is the source of truth. It emits thirteen event types — three lifecycle starts, three lifecycle completions (`ProfileParsed`, `ScoreAssigned`, `ShortlistDecided`), two recruiter decision events, the ERPNext write confirmation, the sanitizer audit events, the approval gate event, and the failure event. `ApplicationView` projects every event into a row used by the UI and the fairness scanner. `ShortlistingWorkflow` both reads and writes on the entity across five steps.

`DriftReportEntity` is a secondary aggregate holding the latest `DriftReport` per cohort dimension. It has a narrow write path (only `FairnessDriftWorkflow`) and a narrow read path (`ShortlistEndpoint` for the `/api/drift-reports` response).

## Defence-in-depth governance flow

For any application that lands in the entity log, the resume passed through:

1. **Special-category sanitizer** — every SCORE-phase tool call is intercepted. Protected-attribute values are replaced with `"[REDACTED]"` before the tool body runs. A `SpecialCategoryRedacted` event records each replacement permanently.
2. **ShortlistAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **Recruiter approval gate** — the workflow cannot call ERPNext until a human has explicitly approved or overridden the agent's decision. The recruiter's decision (and any override rationale) is a permanent entity event.
4. **Fairness drift monitor** — population-level shortlisting rate distributions are scanned daily. A ratio exceeding 1.20 between any two cohorts produces a `DriftReport{breached: true}` visible in the Eval Matrix tab.

Each step addresses a distinct risk. The sanitizer does not detect population-level drift; the drift monitor does not sanitize individual calls. The approval gate does not audit field redaction; the sanitizer events do. Removing any one of them opens an explicit gap the others do not silently cover.
