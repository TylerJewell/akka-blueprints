# Architecture — inbound-lead-qualification

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `LeadEndpoint` accepts a lead form POST, writes `LeadSubmitted` onto `LeadEntity`, and starts `LeadPipelineWorkflow` keyed by `"pipeline-" + leadId`. The workflow's first step (`enrichStep`) emits `EnrichStarted`, passes the instruction context through `PiiSanitizer` to remove raw name and email, then calls `LeadAgent` with `TaskDef.taskType(ENRICH_LEAD)` and `phase = ENRICH` metadata. The agent invokes `EnrichTools.lookupFirmographics` and `EnrichTools.fetchTechStack`; every call passes through `SlackGuardrail` first (non-Slack calls are always accepted). Once the agent returns a `FirmographicProfile`, the workflow writes `EnrichmentCompleted` and advances to `qualifyStep`. The QUALIFY task carries `phase = QUALIFY` and produces a `QualificationScore`. Then `notifyStep` runs — the `SlackGuardrail` now accepts `postLeadToSlack` because `QualificationScored` is present. After `NotificationSent` lands, `evalStep` runs `QualificationEvaluator` over `(score, profile, formData)` — no LLM call — and writes `EvaluationRecorded`. `LeadView` projects every event into a read-model row; `LeadEndpoint` serves the read model over REST and SSE.

The graph has exactly one LLM-calling component. `QualificationEvaluator` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Three properties are worth pausing on:

1. **PII is stripped before the agent sees the instruction.** Between the workflow's `enrichStep` and the `LeadAgent.runSingleTask` call, `PiiSanitizer` intercepts the instruction string and replaces the submitter's name and email with pseudonymous tokens. The agent and all enrichment tools only ever see the company domain and those tokens. The original field values remain on `LeadEntity`.

2. **The Slack write is gated at the tool-call boundary.** `SlackGuardrail` reads `LeadEntity.status` and `score.isPresent()` before every `postLeadToSlack` call. If the entity has not yet recorded `QualificationScored`, the call is rejected and a `GuardrailRejected` event is appended. The guardrail does not block `buildSlackMessage` — only the write.

3. **The task boundary IS the dependency contract.** Between `enrichStep` and `qualifyStep`, the workflow writes `EnrichmentCompleted` onto the entity. The next step then reads `profile` from the entity to build the QUALIFY task's instruction context. The agent never sees enrichment-phase context inside the qualify conversation; the typed handoff is the only path information travels.

Agent calls are bounded by per-step timeouts (60 s on enrich / qualify / notify). `evalStep` is synchronous and finishes in milliseconds.

## State machine

Nine states. The interesting paths:

- The happy path walks `SUBMITTED → ENRICHING → ENRICHED → QUALIFYING → QUALIFIED → NOTIFYING → NOTIFIED → EVALUATED`.
- Three failure transitions land in `FAILED`: an agent error during `ENRICHING`, `QUALIFYING`, or `NOTIFYING`. A failed lead's prior data is preserved on the entity — the UI shows the partial state.
- `GuardrailRejected` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to FAILED.

There is no `APPROVED` or `CONTACTED` state. The qualification score and Slack notification are handed off to the sales rep; the rep's action is outside the system boundary. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`LeadEntity` is the source of truth. It emits ten event types — three lifecycle starts, three lifecycle completions, the evaluation, the guardrail audit, the failure, and the initial submission. `LeadView` projects every event into a row used by the UI. `LeadPipelineWorkflow` both reads (`getLead`) and writes (`startEnrich`, `recordProfile`, `startQualify`, `recordScore`, `startNotify`, `recordNotification`, `recordEval`, `recordGuardrailRejection`, `fail`) on the entity. The relationship between `LeadAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any lead that lands in the entity log, the submitted data passed through:

1. **PII sanitizer** — raw name and email are replaced with pseudonymous tokens before the agent's instruction context is formed. The tokens are stable within a lead's lifecycle.
2. **Phase-sequenced agent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **Slack write guardrail** — the `postLeadToSlack` call is blocked until `QualificationScored` is present. A misordered attempt is rejected before any network socket opens; a `GuardrailRejected` event records the violation.
4. **On-decision evaluator** — every qualification score gets a HIGH / MED / LOW confidence rating against tier-by-size, rep-completeness, and score-tier-alignment rules. LOW-confidence leads are flagged in the UI for sales-ops review.

Each mechanism targets a distinct risk surface. The PII sanitizer does not check scoring quality; the guardrail does not check PII; the evaluator does not check Slack timing. Removing any one of them opens a gap the others do not silently cover.
