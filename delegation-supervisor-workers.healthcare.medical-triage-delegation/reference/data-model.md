# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## MedicalCase (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `caseId` | `String` | no | UUID, also the workflow id |
| `question` | `String` | no | The sanitised submitted question |
| `status` | `CaseStatus` | no | Lifecycle state |
| `symptomsAssessment` | `Optional<SymptomsAssessment>` | yes | SymptomsSpecialist output; null until `SymptomsAssessed` |
| `medicationGuidance` | `Optional<MedicationGuidance>` | yes | MedicationsSpecialist output; null until `MedicationChecked` |
| `careRecommendation` | `Optional<CareRecommendation>` | yes | CarePathwaySpecialist output; null until `CarePathwaySet` |
| `response` | `Optional<SynthesisedResponse>` | yes | Merged response; null until `CaseResponded` |
| `failureReason` | `Optional<String>` | yes | Set on `CaseBlocked` or `CaseDegraded` |
| `complianceScore` | `Optional<Integer>` | yes | 1–5; null until `ComplianceChecked` |
| `complianceRationale` | `Optional<String>` | yes | One-line compliance justification |
| `hotlPending` | `boolean` | no | `true` from `HotlFlagged` until `HotlAcknowledged` |
| `createdAt` | `Instant` | no | Case creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `CaseView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record QuestionRequest(String question, String submittedBy) {}
record SpecialistQueries(String symptomsQuery, String medicationsQuery, String carePathwayQuery) {}

record SymptomsAssessment(
    List<String> identifiedSymptoms,
    String differentialSummary,
    String urgencyIndicator,
    Instant assessedAt
) {}

record MedicationGuidance(
    List<String> relevantMedications,
    List<String> interactions,
    List<String> contraindications,
    Instant guidanceAt
) {}

record CareRecommendation(
    String recommendedPathway,
    String rationale,
    List<String> nextSteps,
    Instant recommendedAt
) {}

record SynthesisedResponse(
    String summary,
    SymptomsAssessment symptomsAssessment,
    MedicationGuidance medicationGuidance,
    CareRecommendation careRecommendation,
    String guardrailVerdict,
    Instant synthesisedAt
) {}
```

## Status enum

```java
enum CaseStatus { TRIAGING, IN_REVIEW, RESPONDED, DEGRADED, BLOCKED }
```

## Events

### CaseEntity

| Event | Trigger |
|---|---|
| `CaseCreated` | Workflow creates the case (`createCase`) |
| `SymptomsAssessed` | SymptomsSpecialist returns a `SymptomsAssessment` |
| `MedicationChecked` | MedicationsSpecialist returns a `MedicationGuidance` |
| `CarePathwaySet` | CarePathwaySpecialist returns a `CareRecommendation` |
| `CaseResponded` | TriageCoordinator synthesis passes the guardrail |
| `CaseDegraded` | A specialist timed out; synthesised from partial inputs |
| `CaseBlocked` | Guardrail rejected the synthesised response |
| `HotlFlagged` | Workflow sets `hotlPending = true` after `RESPONDED` transition |
| `HotlAcknowledged` | Compliance reviewer clears the hotl flag |
| `ComplianceChecked` | `ComplianceSampler` recorded a 1–5 safety score |

### CaseQueue

| Event | Trigger |
|---|---|
| `QuestionSubmitted` | `enqueueQuestion(question, submittedBy)` from endpoint or simulator |

Fields: `{ caseId, question, submittedBy, submittedAt }`.
