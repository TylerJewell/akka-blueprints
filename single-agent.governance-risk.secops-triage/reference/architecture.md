# Architecture — secops-triage

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `FindingEndpoint` accepts a new finding and writes a `FindingIngested` event onto `FindingEntity`. The `FindingEnricher` Consumer subscribes, attaches asset context and threat-intel signals from the in-process seeded registries, and writes the enriched finding back via `attachEnriched`. The same Consumer starts a `TriageWorkflow` instance. The workflow's `triageStep` calls `VulnerabilityTriageAgent` — the single AutonomousAgent — with a brief instruction text and the enriched finding serialized as a `TaskDef.attachment("finding.json", ...)`. The agent's `before-tool-call` guardrail (`RemediationGuardrail`) intercepts any remediation tool call before it executes: it creates an `ApprovalEntity` record, transitions `FindingEntity` to `PENDING_APPROVAL`, and blocks the call. The workflow's `approvalGateStep` then polls `ApprovalEntity` until the analyst approves or rejects via `FindingEndpoint`. On approval, `remediationStep` marks the finding remediated. A parallel `DriftCheckWorkflow` runs every 6 hours, invokes `DriftEvaluator` over recent `FindingView` rows, and emits a `DriftAlertRaised` event if the severity distribution has shifted beyond the calibration threshold. `FindingView` projects every entity event into a read-model row; `FindingEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The drift evaluator (`DriftEvaluator`) is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1): a critical finding is ingested, enriched, triaged, the guardrail intercepts the remediation tool call, and the analyst approves. Two distinct pauses occur:

1. The `FindingEnricher` subscription lag between `FindingIngested` and `FindingEnriched` — sub-second in normal operation (enrichment is in-process lookups against seeded data).
2. The `awaitEnrichedStep` polling loop — polls `FindingEntity` every 1 s up to its 15 s timeout, advancing as soon as `finding.enriched().isPresent()` is true.
3. The `approvalGateStep` pause — the workflow polls `ApprovalEntity` every 2 s waiting for the analyst's decision. This step has a 30-minute timeout; an unresponded approval causes the finding to transition to `FAILED`.

The agent call itself is bounded by `triageStep`'s 60 s timeout.

## State machine

Ten states. The interesting paths:

- The happy path for a high or critical finding is `INGESTED → ENRICHED → TRIAGING → VERDICT_RECORDED → PENDING_APPROVAL → REMEDIATED`.
- Medium and low findings skip the approval gate: `VERDICT_RECORDED → MONITORED` or `VERDICT_RECORDED → ACCEPTED`.
- An analyst rejection lands the finding in `REMEDIATION_REJECTED`. The raw finding and triage verdict are preserved on the entity; the analyst can re-ingest with a corrected action if needed.
- Two failure transitions land in `FAILED`: an enricher error during `INGESTED`, and an agent error (or approval timeout) during `TRIAGING` or `PENDING_APPROVAL`.
- There is no `CLOSED` state. The terminal states are `REMEDIATED`, `REMEDIATION_REJECTED`, `MONITORED`, `ACCEPTED`, and `FAILED` — all of which are visible in the UI list.

## Entity model

`FindingEntity` is the primary source of truth; it emits eleven event types. `ApprovalEntity` is a companion entity tracking only the analyst's approve/reject decision — keeping approval state out of the finding entity simplifies the approval polling loop. `FindingView` projects every `FindingEntity` event into a row used by the UI. `FindingEnricher` subscribes to entity events to compute the enriched form. `TriageWorkflow` both reads (`getFinding`) and writes (`attachEnriched`, `recordVerdict`, `requestApproval`, `markRemediated`, `fail`) on `FindingEntity`, and reads (`getApproval`) and initiates on `ApprovalEntity`. `DriftEvaluator` reads from `FindingView` (not from entity events) so drift detection does not add write pressure to individual entity logs.

## Defence-in-depth governance flow

For any verdict that executes a remediation, the finding passed through:

1. **FindingEnricher** — the model sees asset criticality, existing mitigations, and threat-intel signals rather than the raw finding text alone; richer context reduces severity miscalibration.
2. **VulnerabilityTriageAgent** — one model call, one structured output.
3. **before-tool-call guardrail** — any proposed remediation action is intercepted before execution; no automated change can reach production infrastructure without passing through the approval gate.
4. **Analyst approval gate** — a practitioner reviews the proposed action in context and decides; the workflow cannot advance without an explicit `GRANTED` event.
5. **Periodic drift evaluator** — runs independently every 6 hours; flags systematic scoring drift so analysts know when to re-calibrate the triage baseline.

Each layer is independent. Removing one opens an explicit gap the others do not silently cover.
