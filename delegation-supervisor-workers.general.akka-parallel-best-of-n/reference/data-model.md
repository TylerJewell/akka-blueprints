# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## TranslationJob (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `jobId` | `String` | no | UUID, also the workflow id |
| `sourceText` | `String` | no | The submitted text to translate |
| `targetLanguage` | `String` | no | Target language name (e.g., "Spanish") |
| `status` | `JobStatus` | no | Lifecycle state |
| `variants` | `Optional<List<TranslationVariant>>` | yes | Worker outputs; null until `VariantCollected` first fires |
| `selection` | `Optional<SelectionResult>` | yes | Supervisor selection; null until `JobSelected` |
| `failureReason` | `Optional<String>` | yes | Set on `JobBlocked` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `SelectionEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Job creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `TranslationView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record JobRequest(String sourceText, String targetLanguage, String requestedBy) {}

record VariantInstruction(String register, String instruction) {}
record VariantPlan(List<VariantInstruction> instructions) {}

record TranslationVariant(String register, String translatedText, Instant producedAt) {}

record SelectionResult(
    TranslationVariant selectedVariant,
    String selectionRationale,
    String guardrailVerdict,
    Instant selectedAt
) {}
```

## Status enum

```java
enum JobStatus { PENDING, IN_PROGRESS, SELECTED, PARTIAL, BLOCKED }
```

## Events

### TranslationJobEntity

| Event | Trigger |
|---|---|
| `JobCreated` | Workflow creates the job (`createJob`) |
| `BranchesStarted` | Workflow begins the parallel fan-out after receiving the VariantPlan |
| `VariantCollected` | One worker branch returns a `TranslationVariant`; fired up to three times |
| `JobSelected` | Supervisor selection passes the guardrail |
| `JobPartial` | One or more branches timed out; supervisor selected from remaining variants |
| `JobBlocked` | Guardrail rejected the selected translation |
| `SelectionEvalScored` | `EvalSampler` recorded a 1–5 score |

### JobQueue

| Event | Trigger |
|---|---|
| `JobSubmitted` | `enqueueJob(sourceText, targetLanguage, requestedBy)` from endpoint or simulator |

Fields: `{ jobId, sourceText, targetLanguage, requestedBy, submittedAt }`.
