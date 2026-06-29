# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A question enters through `TriageEndpoint` (`POST /api/triage`) or is dripped by `CaseSimulator` every 60 seconds. Either path writes a `QuestionSubmitted` event onto `CaseQueue`. `CaseRequestConsumer` subscribes to those events and starts one `TriageWorkflow` per submission, keyed by `caseId`.

The workflow is the supervisor. It first runs a deterministic `sanitizeStep` to strip PII and special-category health data from the raw question. It then asks `TriageCoordinator` to classify the question into three specialist queries, fans the work out to `SymptomsSpecialist`, `MedicationsSpecialist`, and `CarePathwaySpecialist` in parallel, then asks `TriageCoordinator` to synthesise. Every transition is written as a command to `CaseEntity`, whose events project into `CaseView`. The endpoint reads and streams the view; `ComplianceSampler` reads it to pick a case to score.

## Interaction sequence

The sequence diagram traces the primary journey: sanitize, classify, parallel three-way fan-out, join, synthesise, guardrail, persist, hotl flag. The `par` block is the delegation-supervisor-workers core — all three specialists run concurrently and the workflow joins their results before synthesis. The guardrail decides between the `RESPONDED` and `BLOCKED` terminals.

## State machine

`MedicalCase` moves `TRIAGING → IN_REVIEW`, then to one of three terminals: `RESPONDED` (guardrail passed), `DEGRADED` (a specialist timed out), or `BLOCKED` (guardrail failed). `RESPONDED` accepts two further self-transitions: `ComplianceChecked` when `ComplianceSampler` records a safety score, and `HotlAcknowledged` when a compliance reviewer clears the pending-review flag.

## Entity model

`CaseQueue` seeds one `MedicalCase` per question. A case owns at most one `SymptomsAssessment`, one `MedicationGuidance`, one `CareRecommendation`, and one `SynthesisedResponse`. The view row mirrors the case with `Optional<T>` on every field that is null before its lifecycle transition (Lesson 6). The `hotlPending` boolean is non-nullable and starts as `false`; it flips to `true` on `HotlFlagged` and back to `false` on `HotlAcknowledged`.
