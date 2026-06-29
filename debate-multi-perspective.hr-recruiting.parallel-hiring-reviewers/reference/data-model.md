# Data model — parallel-hiring-reviewers

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `CandidateProfile` | `candidateName` | `String` | no | The candidate's name. |
| | `roleApplied` | `String` | no | The role the candidate applied for. |
| | `resumeText` | `String` | no | The raw résumé text (redacted before persistence; never stored). |
| | `submittedBy` | `String` | no | UI identifier of the submitter. |
| `EvaluationPlan` | `hrFocus` | `String` | no | Coordinator's brief for the HR axis. |
| | `managerFocus` | `String` | no | Coordinator's brief for the manager axis. |
| | `teamFocus` | `String` | no | Coordinator's brief for the team axis. |
| `EvaluationFinding` | `severity` | `String` | no | `INFO` / `MINOR` / `CONCERN` / `DISQUALIFYING`. |
| | `comment` | `String` | no | The concrete finding text. |
| `PerspectiveReview` | `perspective` | `String` | no | `HR` / `MANAGER` / `TEAM`. |
| | `decision` | `String` | no | `ADVANCE` / `HOLD` / `DECLINE`. |
| | `score` | `int` | no | 1–5 perspective score. |
| | `findings` | `List<EvaluationFinding>` | no | 1–5 findings for this perspective. |
| | `reviewedAt` | `Instant` | no | When the reviewer completed. |
| `HiringRecommendation` | `decision` | `String` | no | `HIRE` / `FURTHER_REVIEW` / `REJECT`. |
| | `rationale` | `String` | no | 60–120 word aggregation explanation. |
| | `perspectiveReviews` | `List<PerspectiveReview>` | no | The perspective reviews aggregated. |
| | `guardrailVerdict` | `String` | no | `"ok"` or `"blocked: <reason>"`. |
| | `decidedAt` | `Instant` | no | When the coordinator aggregated. |
| `FairnessVerdict` | `score` | `int` | no | 1–5 cross-perspective proportionality score. |
| | `rationale` | `String` | no | One-sentence reason. |
| `RedactionResult` | `redactedText` | `String` | no | Résumé text after protected-attribute redaction. |
| | `redactionCount` | `int` | no | Number of substitutions made. |
| `CandidateEvaluation` (entity state, View row source) | `evaluationId` | `String` | no | Unique id. |
| | `candidateName` | `String` | no | Candidate's name. |
| | `roleApplied` | `String` | no | Role applied for. |
| | `status` | `EvaluationStatus` | no | See enum. |
| | `redactedResumeText` | `Optional<String>` | yes | Populated after ProfileSanitized. |
| | `redactionCount` | `Optional<Integer>` | yes | Populated after ProfileSanitized. |
| | `hrReview` | `Optional<PerspectiveReview>` | yes | Populated after HrReviewAttached. |
| | `managerReview` | `Optional<PerspectiveReview>` | yes | Populated after ManagerReviewAttached. |
| | `teamReview` | `Optional<PerspectiveReview>` | yes | Populated after TeamReviewAttached. |
| | `recommendation` | `Optional<HiringRecommendation>` | yes | Populated after RecommendationDecided. |
| | `failureReason` | `Optional<String>` | yes | Populated on EvaluationDegraded / EvaluationBlocked. |
| | `fairnessScore` | `Optional<Integer>` | yes | Populated on FairnessScored. |
| | `fairnessRationale` | `Optional<String>` | yes | Populated on FairnessScored. |
| | `createdAt` | `Instant` | no | When EvaluationCreated emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the evaluation reached a terminal state. |

Every nullable lifecycle field is `Optional<T>` because the View materializer rejects a row record with a non-optional `null` field (Lesson 6).

## Enums

`EvaluationStatus`: `INTAKE`, `EVALUATING`, `DECIDED`, `DEGRADED`, `BLOCKED`.

## Events (`CandidateEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `EvaluationCreated` | `evaluationId, candidateName, roleApplied, createdAt` | Workflow `createStep` | → INTAKE |
| `ProfileSanitized` | `redactedResumeText, redactionCount` | Workflow `sanitizeStep`, after `SpecialCategorySanitizer.redact` | → EVALUATING |
| `HrReviewAttached` | `perspectiveReview` | `HrReviewer` returned | (no status change; populates `hrReview`) |
| `ManagerReviewAttached` | `perspectiveReview` | `ManagerReviewer` returned | (no status change; populates `managerReview`) |
| `TeamReviewAttached` | `perspectiveReview` | `TeamReviewer` returned | (no status change; populates `teamReview`) |
| `RecommendationDecided` | `recommendation` | Coordinator aggregated; guardrail OK | → DECIDED, `finishedAt = now` |
| `EvaluationDegraded` | `recommendation(partial), failureReason` | A reviewer timed out | → DEGRADED, `finishedAt = now` |
| `EvaluationBlocked` | `recommendation(rejected), failureReason` | Output guardrail rejected | → BLOCKED, `finishedAt = now` |
| `FairnessScored` | `score, rationale` | `FairnessSampler` → `FairnessJudge` | (no status change; populates `fairnessScore` + `fairnessRationale`) |

## Events (`CandidateQueue`)

| Event | Payload |
|---|---|
| `ProfileReceived` | `evaluationId, candidateName, roleApplied, resumeText, submittedBy, submittedAt` |

The `resumeText` carried on `ProfileReceived` is the raw text; it is handed to the workflow's start command and redacted in `sanitizeStep` before it touches `CandidateEntity`. The queue event is the only place raw text is held, and it exists for replay/audit; deployers who must avoid even that retention can drop the `resumeText` from the queue event and re-fetch from source.

## View row

`EvaluationRow` mirrors `CandidateEvaluation` minus the heavy `redactedResumeText` (the row keeps `redactionCount`, the three perspective decisions/scores, the overall recommendation, and the fairness score; it truncates each perspective's `findings` to a count to keep the SSE stream small). The UI fetches the full evaluation by id on expand.
