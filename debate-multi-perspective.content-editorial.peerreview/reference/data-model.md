# Data model — peerreview

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `DocumentSubmission` | `title` | `String` | no | The document's title. |
| | `body` | `String` | no | The raw document text (redacted before persistence; never stored). |
| | `submittedBy` | `String` | no | UI identifier of the submitter. |
| `ReviewPlan` | `technicalFocus` | `String` | no | Moderator's brief for the technical axis. |
| | `styleFocus` | `String` | no | Moderator's brief for the style axis. |
| | `complianceFocus` | `String` | no | Moderator's brief for the compliance axis. |
| `ReviewFinding` | `severity` | `String` | no | `INFO` / `MINOR` / `MAJOR` / `BLOCKER`. |
| | `comment` | `String` | no | The concrete finding text. |
| `AxisReview` | `axis` | `String` | no | `TECHNICAL` / `STYLE` / `COMPLIANCE`. |
| | `verdict` | `String` | no | `PASS` / `REVISE` / `FAIL`. |
| | `score` | `int` | no | 1–5 axis score. |
| | `findings` | `List<ReviewFinding>` | no | 1–5 findings for this axis. |
| | `reviewedAt` | `Instant` | no | When the reviewer completed. |
| `OverallVerdict` | `verdict` | `String` | no | `APPROVE` / `REVISE` / `REJECT`. |
| | `summary` | `String` | no | 60–120 word reconciliation. |
| | `axisReviews` | `List<AxisReview>` | no | The axis reviews reconciled. |
| | `guardrailVerdict` | `String` | no | `"ok"` or `"blocked: <reason>"`. |
| | `synthesisedAt` | `Instant` | no | When the moderator reconciled. |
| `ConsistencyVerdict` | `score` | `int` | no | 1–5 cross-axis consistency score. |
| | `rationale` | `String` | no | One-sentence reason. |
| `RedactionResult` | `redactedBody` | `String` | no | Body after PII redaction. |
| | `redactionCount` | `int` | no | Number of substitutions made. |
| `Review` (entity state, View row source) | `reviewId` | `String` | no | Unique id. |
| | `title` | `String` | no | Document title. |
| | `status` | `ReviewStatus` | no | See enum. |
| | `redactedBody` | `Optional<String>` | yes | Populated after DocumentSanitized. |
| | `redactionCount` | `Optional<Integer>` | yes | Populated after DocumentSanitized. |
| | `technical` | `Optional<AxisReview>` | yes | Populated after TechnicalReviewAttached. |
| | `style` | `Optional<AxisReview>` | yes | Populated after StyleReviewAttached. |
| | `compliance` | `Optional<AxisReview>` | yes | Populated after ComplianceReviewAttached. |
| | `verdict` | `Optional<OverallVerdict>` | yes | Populated after VerdictSynthesised. |
| | `failureReason` | `Optional<String>` | yes | Populated on ReviewDegraded / ReviewBlocked. |
| | `consistencyScore` | `Optional<Integer>` | yes | Populated on ConsistencyScored. |
| | `consistencyRationale` | `Optional<String>` | yes | Populated on ConsistencyScored. |
| | `createdAt` | `Instant` | no | When ReviewCreated emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the review reached a terminal state. |

Every nullable lifecycle field is `Optional<T>` because the View materializer rejects a row record with a non-optional `null` field (Lesson 6).

## Enums

`ReviewStatus`: `INTAKE`, `REVIEWING`, `SYNTHESISED`, `DEGRADED`, `BLOCKED`.

## Events (`ReviewEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `ReviewCreated` | `reviewId, title, createdAt` | Workflow `createStep` | → INTAKE |
| `DocumentSanitized` | `redactedBody, redactionCount` | Workflow `sanitizeStep`, after `PiiSanitizer.redact` | → REVIEWING |
| `TechnicalReviewAttached` | `axisReview` | `TechnicalReviewer` returned | (no status change; populates `technical`) |
| `StyleReviewAttached` | `axisReview` | `StyleReviewer` returned | (no status change; populates `style`) |
| `ComplianceReviewAttached` | `axisReview` | `ComplianceReviewer` returned | (no status change; populates `compliance`) |
| `VerdictSynthesised` | `verdict` | Moderator reconciled; guardrail OK | → SYNTHESISED, `finishedAt = now` |
| `ReviewDegraded` | `verdict(partial), failureReason` | A reviewer timed out | → DEGRADED, `finishedAt = now` |
| `ReviewBlocked` | `verdict(rejected), failureReason` | Output guardrail rejected | → BLOCKED, `finishedAt = now` |
| `ConsistencyScored` | `score, rationale` | `EvalSampler` → `EvalJudge` | (no status change; populates `consistencyScore` + `consistencyRationale`) |

## Events (`SubmissionQueue`)

| Event | Payload |
|---|---|
| `SubmissionReceived` | `reviewId, title, body, submittedBy, submittedAt` |

The `body` carried on `SubmissionReceived` is the raw text; it is handed to the workflow's start command and redacted in `sanitizeStep` before it touches `ReviewEntity`. The queue event is the only place raw text is held, and it exists for replay/audit; deployers who must avoid even that retention can drop the `body` from the queue event and re-fetch from source.

## View row

`ReviewRow` mirrors `Review` minus the heavy `redactedBody` text (the row keeps `redactionCount`, the three axis verdicts/scores, the overall verdict, and the consistency score; it truncates each axis's `findings` to a count to keep the SSE stream small). The UI fetches the full review by id on expand.
