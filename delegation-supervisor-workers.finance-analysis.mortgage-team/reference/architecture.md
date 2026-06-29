# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

An application enters through `MortgageEndpoint` (`POST /api/applications`) or is dripped by `ApplicationSimulator` every 90 seconds. Either path writes an `ApplicationSubmitted` event onto `ApplicationQueue`. `ApplicationConsumer` subscribes to those events and starts one `MortgageWorkflow` per submission, keyed by `applicationId`.

The workflow is the supervisor. It asks `MortgageSupervisor` to decompose the application into three parallel work items, fans those out to `EligibilityAgent`, `DocumentationAgent`, and `UnderwritingAgent` concurrently, then collects their typed assessments. `CustomerCommsAgent` drafts an acknowledgement notification after the fan-out starts. The Supervisor merges all three assessments into a `RecommendationPacket`. A deterministic compliance step vets the proposed terms. If the terms are compliant, the workflow pauses at the HITL gate and waits for a human underwriter's decision. Every lifecycle transition is written as a command to `ApplicationEntity`, whose events project into `ApplicationView`. `MortgageEndpoint` reads and streams the view; `DecisionEvalSampler` reads it to pick a decided application to score.

## Interaction sequence

The sequence diagram traces the primary journey: decompose, parallel fan-out (three agents simultaneously), join, merge, compliance check, HITL pause, human decision, outcome communications, persist. The `par` block is the delegation-supervisor-workers core — all three workers run concurrently and the workflow joins their results. The compliance step and the HITL gate are shown as explicit decision points, making it clear that no automated approval can occur without the underwriter's call to `POST /api/applications/{id}/decision`.

## State machine

`MortgageApplication` moves `RECEIVED → UNDER_REVIEW`, then forks to one of five terminals: `PENDING_SIGNOFF` (compliance passed; waiting for HITL), `COMPLIANCE_HOLD` (sectoral filter blocked), `REVIEW_REQUIRED` (a worker timed out). From `PENDING_SIGNOFF` it moves to either `APPROVED` or `DECLINED` on the human's decision. `APPROVED` accepts one further `DecisionEvalScored` self-transition when `DecisionEvalSampler` records a quality score.

## Entity model

`ApplicationQueue` seeds one `MortgageApplication` per submission. A mortgage application owns at most one `RecommendationPacket`, which itself contains exactly one `EligibilityAssessment`, one `DocumentationAssessment`, and one `UnderwritingAssessment`. The `UnderwritingAssessment` owns a `TermStructure`. The `ApplicationView` row mirrors the application with `Optional<T>` on every field that is null before its lifecycle transition, per Lesson 6, so the view is safe to stream from the earliest `RECEIVED` state onwards.
