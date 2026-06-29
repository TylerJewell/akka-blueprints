# Data model — scored-loop-tailor

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `JobPosting` | `jobTitle` | `String` | no | The role title submitted by the user. |
| | `description` | `String` | no | Full job description text; may be empty string. |
| | `candidateProfile` | `String` | no | Optional background summary; may be empty string. |
| `CvDraft` | `text` | `String` | no | The full CV text including all section headings. |
| | `sectionsPresent` | `List<String>` | no | Top-level section headings found in `text` (e.g. `["Summary","Experience","Skills"]`). |
| | `draftedAt` | `Instant` | no | When the Tailor returned the draft. |
| `SectionGuardrailVerdict` | `passed` | `boolean` | no | Whether the guardrail accepted the draft. |
| | `reasonCode` | `String` | no | `OK` or `MISSING_SECTIONS`. |
| | `detail` | `String` | no | Human-readable detail; empty when `passed=true`. |
| `ReviewNotes` | `bullets` | `List<String>` | no | 0–3 short bullets (empty on `APPROVE`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `CvReview` | `verdict` | `ReviewVerdict` | no | `APPROVE` or `REVISE`. |
| | `notes` | `ReviewNotes` | no | See above. |
| | `score` | `int` | no | 1–5 rubric minimum across four dimensions. |
| | `reviewedAt` | `Instant` | no | When the Reviewer returned. |
| `CvAttempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop (includes guardrail-blocked attempts). |
| | `draft` | `CvDraft` | no | The Tailor's output for this attempt. |
| | `guardrail` | `SectionGuardrailVerdict` | no | The deterministic check's verdict. |
| | `review` | `Optional<CvReview>` | yes | Empty when the guardrail blocked; populated otherwise. |
| `Application` (entity state) | `applicationId` | `String` | no | Unique id. |
| | `jobTitle` | `String` | no | Original job title. |
| | `description` | `String` | no | Original job description. |
| | `maxAttempts` | `int` | no | Per-application retry ceiling (default 4). |
| | `status` | `ApplicationStatus` | no | See enum. |
| | `attempts` | `List<CvAttempt>` | no | Bounded at `maxAttempts`; starts empty. |
| | `approvedAttemptNumber` | `Optional<Integer>` | yes | Populated on `ApplicationApproved`. |
| | `approvedText` | `Optional<String>` | yes | Populated on `ApplicationApproved`. |
| | `rejectionReason` | `Optional<String>` | yes | Populated on `ApplicationRejectedFinal`. |
| | `createdAt` | `Instant` | no | When `ApplicationCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the application reached a terminal state. |

## Enums

`ApplicationStatus`: `TAILORING`, `REVIEWING`, `APPROVED`, `REJECTED_FINAL`.

`ReviewVerdict`: `APPROVE`, `REVISE`.

## Events (`ApplicationEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `ApplicationCreated` | `jobTitle, description, maxAttempts, createdAt` | Workflow `startStep` | → `TAILORING` |
| `CvDrafted` | `attemptNumber, draft: CvDraft` | After `tailorStep` returns | (no status change; appends to `attempts[]`) |
| `SectionGuardrailVerdictRecorded` | `attemptNumber, verdict: SectionGuardrailVerdict` | After `guardrailStep` | `passed=true` → `REVIEWING`; `passed=false` → `TAILORING` (re-tailor) |
| `CvReviewed` | `attemptNumber, review: CvReview` | After `reviewStep` returns | (no status change; populates `attempts[n].review`) |
| `ApplicationApproved` | `attemptNumber, approvedText` | `CvReview.verdict = APPROVE` | → `APPROVED`, `finishedAt = now` |
| `ApplicationRejectedFinal` | `bestAttemptNumber, bestText, rejectionReason` | `attempts.size() == maxAttempts` AND last review is `REVISE`; OR `defaultStepRecovery` failover | → `REJECTED_FINAL`, `finishedAt = now` |
| `ReviewEvalRecorded` | `attemptNumber, verdict, score, missingSections, recordedAt` | `EvalSampler` per reviewed attempt; workflow on terminal transition | (no status change; appends to an internal `evalEvents[]` view-side projection) |

## Events (`SubmissionQueue`)

| Event | Payload |
|---|---|
| `JobPostingSubmitted` | `applicationId, jobTitle, description, candidateProfile, submittedAt` |

## View row

`ApplicationRow` is structurally identical to `Application` — the `attempts` list is bounded at `maxAttempts` (default 4) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `ApplicationEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllApplications` returning the full list. Callers filter by `status` client-side.
