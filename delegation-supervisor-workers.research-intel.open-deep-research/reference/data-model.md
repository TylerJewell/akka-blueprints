# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## ResearchRun (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `runId` | `String` | no | UUID, also the workflow id |
| `question` | `String` | no | The submitted research question |
| `status` | `RunStatus` | no | Lifecycle state |
| `webContent` | `Optional<WebBundle>` | yes | WebBrowsingAgent output; null until `WebContentAttached` |
| `docContent` | `Optional<DocumentBundle>` | yes | FileReaderAgent output; null until `DocContentAttached` |
| `answer` | `Optional<SynthesisedAnswer>` | yes | Manager synthesis; null until `RunAnswered` |
| `blockedUrl` | `Optional<String>` | yes | First URL blocked by the guardrail, if any |
| `failureReason` | `Optional<String>` | yes | Set on `RunBlocked` or `RunDegraded` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `RunEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `oversightFlagged` | `boolean` | no | True when `OversightFlagged` event has been emitted |
| `createdAt` | `Instant` | no | Run creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `ResearchView` row type `ResearchRunRow` mirrors this with `Optional<T>` on every nullable field. `oversightFlagged` remains a primitive boolean in the row type.

## Supporting records

```java
record QuestionRequest(String question, String requestedBy) {}
record RetrievalPlan(List<String> urlsToFetch, String documentReference) {}
record WebPage(String url, String title, String extractedText, Instant fetchedAt) {}
record WebBundle(List<WebPage> pages, boolean piiSanitized, Instant gatheredAt) {}
record DocumentSection(String heading, String content) {}
record DocumentBundle(String documentRef, List<DocumentSection> sections,
                      boolean piiSanitized, Instant readAt) {}
record Citation(String text, String sourceUrl, String sourceDocument) {}
record SynthesisedAnswer(String summary, List<Citation> citations,
                         boolean piiSanitized, Instant synthesisedAt) {}
```

## Status enum

```java
enum RunStatus { QUEUED, IN_PROGRESS, ANSWERED, DEGRADED, BLOCKED }
```

## Events

### ResearchRunEntity

| Event | Trigger |
|---|---|
| `RunCreated` | Workflow creates the run (`createRun`) |
| `WebContentAttached` | WebBrowsingAgent returns a `WebBundle` |
| `DocContentAttached` | FileReaderAgent returns a `DocumentBundle` |
| `RunAnswered` | Manager synthesis completes; stored as `SynthesisedAnswer` |
| `RunDegraded` | A worker timed out; synthesis ran on partial input |
| `RunBlocked` | All URLs were blocked and no document content came back |
| `RunEvalScored` | `EvalSampler` recorded a 1–5 score and rationale |
| `OversightFlagged` | Workflow detected a retry-budget breach |

### QuestionQueue

| Event | Trigger |
|---|---|
| `QuestionSubmitted` | `enqueueQuestion(question, requestedBy)` from endpoint or simulator |

Fields: `{ runId, question, requestedBy, submittedAt }`.
