# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## AnalysisJob (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `jobId` | `String` | no | UUID, also the workflow id |
| `question` | `String` | no | The question submitted with the image batch |
| `imageRefs` | `List<String>` | no | Ordered list of image identifiers from the submission |
| `status` | `JobStatus` | no | Lifecycle state |
| `sanitizedImages` | `Optional<List<SanitizedImage>>` | yes | Post-sanitization payloads; null until `ImagesSanitized` |
| `reports` | `Optional<List<ImageReport>>` | yes | Per-image analyst outputs; null until at least one `ReportAttached` |
| `synthesised` | `Optional<SynthesisedAnswer>` | yes | Merged answer; null until `JobSynthesised` |
| `failureReason` | `Optional<String>` | yes | Set on `JobBlocked` or `JobPartial` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `JobEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Job creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `AnalysisView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record BatchSubmission(List<ImagePayload> images, String question, String submittedBy) {}
record ImagePayload(String imageRef, String base64Data, String mimeType) {}
record SanitizedImage(String imageRef, String sanitizedBase64Data, String mimeType,
                      boolean piiDetected) {}
record ImageTask(String imageRef, String instruction) {}
record DetectedLabel(String label, double confidence) {}
record ImageReport(String imageRef, String description, List<String> tags,
                   List<DetectedLabel> detectedLabels, Instant analysedAt) {}
record SynthesisedAnswer(String answer, List<ImageReport> imageSummaries,
                         String guardrailVerdict, Instant synthesisedAt) {}
```

## Status enum

```java
enum JobStatus { QUEUED, ANALYSING, SYNTHESISED, PARTIAL, BLOCKED }
```

## Events

### AnalysisJobEntity

| Event | Trigger |
|---|---|
| `JobCreated` | Workflow creates the job (`createJob`) with the image refs and question |
| `ImagesSubmitted` | Raw images recorded on the entity before sanitization |
| `ImagesSanitized` | `PiiSanitizer` has processed all images (`markSanitized`) |
| `AnalysisStarted` | `OpusCoordinator.DECOMPOSE` returned a task list; sub-agents about to run |
| `ReportAttached` | One `HaikuImageAnalyst` returned an `ImageReport` (`attachReport`) |
| `JobSynthesised` | Coordinator synthesis passed the guardrail (`synthesise`) |
| `JobPartial` | One or more sub-agents timed out; synthesised from partial reports (`markPartial`) |
| `JobBlocked` | Guardrail rejected the synthesised answer (`block`) |
| `JobEvalScored` | `EvalSampler` recorded a 1–5 score (`recordEval`) |

### BatchQueue

| Event | Trigger |
|---|---|
| `BatchSubmitted` | `enqueueBatch(images, question, submittedBy)` from endpoint or simulator |

Fields: `{ jobId, imageCount, question, submittedBy, submittedAt }`.
