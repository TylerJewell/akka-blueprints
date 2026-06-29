# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## CandidateApplication (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `applicationId` | `String` | no | UUID, also the workflow id |
| `candidateId` | `String` | no | Candidate identifier from the submitter |
| `roleId` | `String` | no | Role being applied for |
| `status` | `ApplicationStatus` | no | Lifecycle state |
| `screeningReport` | `Optional<ScreeningReport>` | yes | ScreeningAgent output; null until `ScreeningReportAttached` |
| `schedulingProposal` | `Optional<SchedulingProposal>` | yes | SchedulingAgent output; null until `SchedulingProposalAttached` |
| `recommendation` | `Optional<HiringRecommendation>` | yes | Consolidated recommendation; null until `ApplicationRecommended` or `ApplicationRejected` |
| `failureReason` | `Optional<String>` | yes | Set on `ApplicationDegraded` or `ApplicationPolicyHeld` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `EvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Application creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `ApplicationView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record ApplicationRequest(String candidateId, String roleId, String resumeText,
                          String availabilityWindow, String requestedBy) {}

record SanitizedApplication(String candidateId, String roleId, String resumeText,
                            String availabilityWindow, boolean sanitized) {}

record DelegationPlan(String screeningQuery, String schedulingContext) {}

record ProposedSlot(String slotId, Instant startTime, Instant endTime, String interviewFormat) {}

record ScreeningReport(String candidateId, int qualificationScore,
                       List<String> strengthHighlights, List<String> gapNotes,
                       Instant screenedAt) {}

record SchedulingProposal(String candidateId, List<ProposedSlot> slots, Instant proposedAt) {}

record HiringRecommendation(String decision, int screeningScore,
                            List<ProposedSlot> proposedSlots, String rationale,
                            String guardrailVerdict, Instant consolidatedAt) {}
```

## Status enum

```java
enum ApplicationStatus { RECEIVED, UNDER_REVIEW, RECOMMENDED, REJECTED, DEGRADED, POLICY_HOLD }
```

## Events

### ApplicationEntity

| Event | Trigger |
|---|---|
| `ApplicationReceived` | Workflow creates the application (`receiveApplication`) |
| `ReviewStarted` | Workflow transitions to UNDER_REVIEW after DelegationPlan is ready |
| `ScreeningReportAttached` | ScreeningAgent returns a `ScreeningReport` |
| `SchedulingProposalAttached` | SchedulingAgent returns a `SchedulingProposal` |
| `ApplicationRecommended` | Supervisor consolidation decision is RECOMMENDED and guardrail passes |
| `ApplicationRejected` | Supervisor consolidation decision is REJECTED and guardrail passes |
| `ApplicationDegraded` | A worker timed out; consolidated from partial input |
| `ApplicationPolicyHeld` | Before-invocation guardrail blocked a delegate call |
| `EvalScored` | `EvalSampler` recorded a 1–5 score |

### ApplicationQueue

| Event | Trigger |
|---|---|
| `ApplicationSubmitted` | `submitApplication(candidateId, roleId, requestedBy)` from endpoint or simulator |

Fields: `{ applicationId, candidateId, roleId, requestedBy, submittedAt }`.
