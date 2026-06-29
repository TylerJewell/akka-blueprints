# Data model — cross-agency-case-handoff-mesh

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `InboundCase` | `caseId` | `String` | no | Server-assigned UUID. |
| | `constituentId` | `String` | no | Opaque constituent identifier from the submission channel. Dropped at every cross-boundary handoff. |
| | `programType` | `String` | no | `"food-assistance"` / `"housing-aid"` / `"child-services"` / `"income-support"` / other. |
| | `geographicRegion` | `String` | no | ISO 3166-2 subdivision code (e.g. `"US-CA"`). |
| | `summary` | `String` | no | Short human-readable case summary; may contain PII. |
| | `fullPayload` | `String` | no | Raw case payload; may contain SSN, DOB, income figures. Never forwarded cross-boundary. |
| | `submittedAt` | `Instant` | no | When the submission channel delivered the case. |
| `ScopedCase` | `caseId` | `String` | no | Same as `InboundCase.caseId`. |
| | `programType` | `String` | no | Passed through unmodified (not PII). |
| | `geographicRegion` | `String` | no | Passed through unmodified. |
| | `redactedSummary` | `String` | no | PII redacted from `summary`. |
| | `redactedPayload` | `String` | no | PII redacted from `fullPayload`. |
| | `droppedFieldCategories` | `List<String>` | no | Field categories withheld, e.g. `["ssn","date-of-birth","income-figure","address"]`. |
| `RoutingDecision` | `segment` | `CaseSegment` | no | `INTAKE` / `BENEFITS` / `UNROUTABLE`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `rationale` | `String` | no | One short sentence. |
| `JurisdictionVerdict` | `allowed` | `boolean` | no | `true` to proceed with agent invocation; `false` to block. |
| | `violations` | `List<String>` | no | Empty when `allowed=true`. Jurisdiction rule tokens otherwise. |
| | `agencyCode` | `String` | no | `"INTAKE-01"` or `"BENEFITS-01"`. |
| | `policyVersion` | `String` | no | `"v1"`. |
| `SegmentOutcome` | `segment` | `CaseSegment` | no | `INTAKE` / `BENEFITS`. |
| | `determination` | `DeterminationType` | no | See enum. |
| | `summaryNote` | `String` | no | 2–4 sentences. |
| | `agencyTag` | `String` | no | `"intake"` / `"benefits"`. |
| | `completedAt` | `Instant` | no | When the agency agent finished. |
| `HandoffApproval` | `approvedBy` | `String` | no | Operator identifier. |
| | `note` | `String` | no | Free-text note recorded for audit. |
| | `approvedAt` | `Instant` | no | When the operator approved. |
| `HandoffRejection` | `rejectedBy` | `String` | no | Operator identifier. |
| | `reason` | `String` | no | Rejection reason. |
| | `rejectedAt` | `Instant` | no | When the operator rejected. |
| `Case` (entity state) | `caseId` | `String` | no | — |
| | `inbound` | `InboundCase` | no | Captured once at create. |
| | `scoped` | `Optional<ScopedCase>` | yes | Populated after `CaseScoped`. |
| | `routing` | `Optional<RoutingDecision>` | yes | Populated after `CaseRouted`. |
| | `jurisdictionVerdict` | `Optional<JurisdictionVerdict>` | yes | Populated after `JurisdictionChecked`. |
| | `intakeOutcome` | `Optional<SegmentOutcome>` | yes | Populated after `SegmentCompleted` for intake. |
| | `benefitsOutcome` | `Optional<SegmentOutcome>` | yes | Populated after `SegmentCompleted` for benefits. |
| | `handoffApproval` | `Optional<HandoffApproval>` | yes | Populated after `HandoffEmitted`. |
| | `handoffRejection` | `Optional<HandoffRejection>` | yes | Populated after `HandoffRejected`. |
| | `status` | `CaseStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `CaseRegistered` emitted. |
| | `closedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6.

## Enums

`CaseSegment`: `INTAKE`, `BENEFITS`, `UNROUTABLE`.

`DeterminationType`: `ELIGIBLE`, `INELIGIBLE`, `REFERRED_TO_BENEFITS`, `AWARD_ISSUED`, `AWARD_DENIED`, `PENDING_DOCUMENTS`, `ESCALATED`.

`CaseStatus`: `RECEIVED`, `SCOPED`, `ROUTED`, `JURISDICTION_CHECK`, `IN_SEGMENT`, `SEGMENT_COMPLETE`, `HANDOFF_PENDING`, `HANDOFF_EMITTED`, `HANDOFF_REJECTED`, `JURISDICTION_BLOCKED`, `CLOSED`, `UNROUTABLE`.

## Events (`CaseEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `CaseRegistered` | `inbound` | `PiiScopingFilter.registerInbound` | → `RECEIVED` |
| `CaseScoped` | `scoped` | `PiiScopingFilter.attachScoped` | → `SCOPED` |
| `CaseRouted` | `routing` | `HandoffWorkflow.routeStep` | → `ROUTED` |
| `JurisdictionChecked` | `jurisdictionVerdict` | `HandoffWorkflow.guardStep` | → `JURISDICTION_CHECK` (then `IN_SEGMENT` if allowed) |
| `SegmentStarted` | `segment` | `HandoffWorkflow.segmentStep` entry | → `IN_SEGMENT` |
| `SegmentCompleted` | `segmentOutcome` | `HandoffWorkflow.segmentStep` result | → `SEGMENT_COMPLETE` |
| `HandoffPending` | — | `HandoffWorkflow.approvalStep` | → `HANDOFF_PENDING` |
| `HandoffEmitted` | `handoffApproval` | operator `approve-handoff` | → `HANDOFF_EMITTED` |
| `HandoffRejected` | `handoffRejection` | operator `reject-handoff` | → `HANDOFF_REJECTED` (terminal) |
| `JurisdictionBlocked` | `jurisdictionVerdict` | `HandoffWorkflow.guardStep` (allowed=false) | → `JURISDICTION_BLOCKED` (terminal) |
| `CaseClosed` | — | `HandoffWorkflow.emitStep` (last segment approved) | → `CLOSED` (terminal) |
| `CaseUnroutable` | `routing.rationale` | `HandoffWorkflow.routeStep` (UNROUTABLE) | → `UNROUTABLE` (terminal) |

## Events (`CaseInbox`)

| Event | Payload |
|---|---|
| `InboundCaseReceived` | `inbound` (the raw, pre-scoping payload — the audit log before any redaction) |

## View row

`CaseRow` is `Case` verbatim. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record without an error. The view's single query is `SELECT * AS cases FROM case_view` with no `WHERE segment` or `WHERE status` filter — callers (`CaseEndpoint`) apply those client-side.
