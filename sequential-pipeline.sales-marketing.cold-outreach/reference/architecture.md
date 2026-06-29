# Architecture — cold-outreach

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence, with an explicit human pause between the second and third tasks. `OutreachEndpoint` accepts a `{contactEmail, companyName}` POST, writes `ProspectCreated` onto `ProspectEntity`, and starts `OutreachPipelineWorkflow` keyed by `"pipeline-" + prospectId`. The workflow's first step (`researchStep`) emits `ResearchStarted`, then calls `OutreachAgent` with `TaskDef.taskType(RESEARCH_PROSPECT)` and a `phase = RESEARCH` metadata tag. The agent invokes `ResearchTools.lookupFirmographics` and `ResearchTools.fetchIntentSignals`; every call passes through `SendGuardrail` first (which acts only on `sendEmail`, so RESEARCH-phase calls pass through). Once the agent returns a `ProspectProfile`, the workflow writes `ProspectResearched` and advances to `draftStep` — same pattern, the DRAFT task carries `phase = DRAFT`. The `ComplianceGuardrail` intercepts every agent response during `draftStep` and checks for the unsubscribe and sender-address sentinels. After `EmailDrafted` lands, the workflow enters `approvalStep`: it emits `ReviewRequested`, transitions the entity to `AWAITING_REVIEW`, and pauses. A human reviewer submits a decision via `POST /api/outreach/{id}/review`. The entity records `ReviewDecided`; the workflow resumes. If approved, `sendStep` runs and the agent calls `SendTools.sendEmail` — which passes through `SendGuardrail` only after confirming that `ReviewDecided{approved: true}` is present on the entity. `ProspectView` projects every event into a read-model row; `OutreachEndpoint` serves it to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `ComplianceGuardrail` is a deterministic rule-based checker with no LLM call; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Three properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `researchStep` and `draftStep`, the workflow writes `ProspectResearched` onto the entity. The next step then reads `profile` from the entity to build the DRAFT task's instruction context. The agent never sees research-phase context inside the draft task's conversation; the typed handoff is the only path information travels.
2. The `approvalStep` pause is not an agent call — it is a workflow-level gate. The workflow emits `ReviewRequested` and suspends on a 24 h timer. The reviewer's POST to `/api/outreach/{id}/review` is the only signal that advances the workflow past this point. Neither the agent nor the guardrails can bypass it; the workflow itself enforces the pause.
3. Every tool call is filtered through `SendGuardrail`. For RESEARCH and DRAFT tasks the guardrail passes all tool calls without inspection (they are not `sendEmail`). For the SEND task, the guardrail reads the entity state and verifies that `ReviewDecided{approved: true}` is present before allowing `sendEmail` to run.

The agent calls are bounded by per-step timeouts (60 s on research / draft, 30 s on send). `approvalStep` has a 24 h timeout enforcing reviewer SLA.

## State machine

Eleven states. The interesting paths:

- The happy path walks `CREATED → RESEARCHING → RESEARCHED → DRAFTING → DRAFTED → AWAITING_REVIEW → SENDING → SENT`.
- A reviewer rejection branches at `AWAITING_REVIEW → REVIEW_REJECTED`. No send, no `EmailSent` event.
- Three failure transitions land in `FAILED`: an agent error during `RESEARCHING`, `DRAFTING`, or `SENDING`. A `FAILED` prospect's prior data is preserved on the entity — the UI shows the partial state.
- `GuardrailRejected` and `ComplianceChecked` are side-events recorded for audit; they do not transition status. Only an exhausted retry budget, a step timeout, or a reviewer rejection changes the pipeline's path.

There is no `PUBLISHED` or `CAMPAIGN_SENT` state. This blueprint handles one prospect per run; batching across a prospect list is a deployer concern outside the scope of the sample.

## Entity model

`ProspectEntity` is the source of truth. It emits twelve event types — research, draft, compliance, review, send lifecycle events plus the guardrail audit and failure events. `ProspectView` projects every event into a row used by the UI. `OutreachPipelineWorkflow` both reads (`getProspect`) and writes (`startResearch`, `recordProfile`, `startDraft`, `recordDraft`, `recordComplianceCheck`, `requestReview`, `recordReviewDecision`, `startSend`, `recordEmail`, `recordGuardrailRejection`, `fail`) on the entity.

## Defence-in-depth governance flow

For any outreach that reaches `SENT`, the email passed through:

1. **Send-gate guardrail (G1)** — every `sendEmail` tool call is blocked unless `ReviewDecided{approved: true}` is recorded on the entity. A misordered call during RESEARCH or DRAFT is caught here.
2. **ComplianceGuardrail (H2)** — every proposed email body returned by the DRAFT task is checked for unsubscribe notice and sender address before it is committed to the entity.
3. **OutreachAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
4. **HITL approval gate (HITL1)** — a human reviewer reads the draft and explicitly approves it before `sendStep` starts. Neither the agent nor the guardrails can move the pipeline past `approvalStep` without this decision.

Each step is independent. The send guardrail does not check compliance formatting; the compliance guardrail does not check whether a review decision exists; the HITL gate does not inspect email content. Removing any one of them opens a gap the other two do not cover.
