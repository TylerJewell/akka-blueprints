# Data model — financial-research-pipeline

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ResearchQuery` | `topic` | `String` | no | The submitted research topic. |
| | `sectorTag` | `String` | no | Financial sector: `equity`, `credit`, `macro`, `commodities`, `general`. |
| | `wordCeiling` | `int` | no | Maximum word count for the final report. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `SubTask` | `taskNumber` | `int` | no | 1-indexed, sequential. |
| | `scope` | `String` | no | One-sentence retrieval objective. |
| | `sectorTag` | `String` | no | Sector for this sub-task. |
| `ResearchPlan` | `subTasks` | `List<SubTask>` | no | Ordered list of sub-tasks. |
| | `planSummary` | `String` | no | Two-sentence decomposition rationale. |
| | `plannedAt` | `Instant` | no | When `PlannerAgent` returned. |
| `SourceExcerpt` | `text` | `String` | no | Retrieved passage. |
| | `provenance` | `String` | no | Citation string. |
| | `relevanceScore` | `double` | no | Float `[0.0, 1.0]`. |
| `SourceBundle` | `taskNumber` | `int` | no | Matches the sub-task. |
| | `excerpts` | `List<SourceExcerpt>` | no | Ranked excerpts, most relevant first. |
| | `retrievedAt` | `Instant` | no | When `SearchAgent` returned. |
| `AnalysisSection` | `sectionNumber` | `int` | no | 1-indexed. |
| | `heading` | `String` | no | 5-word-or-fewer section title. |
| | `claims` | `List<String>` | no | 2–5 factual claims. |
| | `citations` | `List<String>` | no | Provenance strings supporting the claims. |
| `SanitizerLog` | `changed` | `boolean` | no | Whether any item was removed. |
| | `removedItems` | `List<String>` | no | Items removed by the sanitiser (empty when `changed = false`). |
| | `reasonCode` | `String` | no | `SECTOR_SCRUB` (always for this sanitiser). |
| `VerificationNotes` | `bullets` | `List<String>` | no | 0–4 short bullets (empty on `APPROVE`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `Verification` | `verdict` | `VerifierVerdict` | no | `APPROVE` or `REVISE`. |
| | `notes` | `VerificationNotes` | no | See above. |
| | `score` | `int` | no | 1–5 rubric score. |
| | `verifiedAt` | `Instant` | no | When `VerifierAgent` returned. |
| `ReportDraft` | `draftNumber` | `int` | no | 1-indexed; monotonic including guardrail-blocked drafts. |
| | `text` | `String` | no | Prose report text. |
| | `wordCount` | `int` | no | Word count of `text`. |
| | `draftedAt` | `Instant` | no | When `WriterAgent` returned. |
| `Report` (entity state) | `reportId` | `String` | no | Unique id. |
| | `topic` | `String` | no | Original research query. |
| | `sectorTag` | `String` | no | Financial sector. |
| | `wordCeiling` | `int` | no | Maximum word count. |
| | `maxVerifyAttempts` | `int` | no | Per-report verify ceiling (default 3). |
| | `status` | `ReportStatus` | no | See enum. |
| | `plan` | `Optional<ResearchPlan>` | yes | Populated on `PlanRecorded`. |
| | `sourceBundles` | `List<SourceBundle>` | no | One per sub-task; grows during `RESEARCHING`. |
| | `analysisSections` | `List<AnalysisSection>` | no | Populated on `AnalysisSectionsRecorded`. |
| | `sanitizerLog` | `SanitizerLog` | no | Populated on `SanitizerLogRecorded`; default `changed=false`. |
| | `drafts` | `List<ReportDraft>` | no | Bounded at `maxVerifyAttempts`; grows during `WRITING`. |
| | `verifications` | `List<Verification>` | no | Bounded at `maxVerifyAttempts`; grows during `VERIFYING`. |
| | `approvedDraftNumber` | `Optional<Integer>` | yes | Populated on `ReportApprovedPending`. |
| | `approvedText` | `Optional<String>` | yes | Populated on `ReportApproved`. |
| | `rejectionReason` | `Optional<String>` | yes | Populated on `ReportRejectedFinal`. |
| | `createdAt` | `Instant` | no | When `ReportCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the report reached a terminal state. |

## Enums

`ReportStatus`: `PLANNING`, `RESEARCHING`, `ANALYSING`, `WRITING`, `VERIFYING`, `AWAITING_COMPLIANCE`, `APPROVED`, `REJECTED_FINAL`.

`VerifierVerdict`: `APPROVE`, `REVISE`.

## Events (`ResearchEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `ReportCreated` | `topic, sectorTag, wordCeiling, maxVerifyAttempts, createdAt` | Workflow `startStep` | → `PLANNING` |
| `PlanRecorded` | `plan: ResearchPlan` | After `planStep` returns | → `RESEARCHING` |
| `SourceBundleAdded` | `bundle: SourceBundle` | After each `searchStep` returns | (no status change until all sub-tasks complete; final bundle → `ANALYSING`) |
| `AnalysisSectionsRecorded` | `sections: List<AnalysisSection>` | After `analyseStep` returns | (stays in `ANALYSING`; sanitiser runs next) |
| `SanitizerLogRecorded` | `log: SanitizerLog` | After `sanitiseStep` completes | → `WRITING` |
| `DraftWritten` | `draftNumber, draft: ReportDraft` | After `writeStep` returns | (no status change; appends to `drafts[]`) |
| `DraftGuardrailVerdictRecorded` | `draftNumber, passed, reasonCode, detail` | After `guardrailStep` | `passed=true` → `VERIFYING`; `passed=false` → stays `WRITING` (re-write) |
| `DraftVerified` | `draftNumber, verification: Verification` | After `verifyStep` returns | (no status change; appends to `verifications[]`) |
| `ReportApprovedPending` | `draftNumber, approvedText` | `Verification.verdict = APPROVE` | → `AWAITING_COMPLIANCE` |
| `ReportApproved` | `reviewerId, reviewNotes, approvedAt` | Compliance reviewer calls approve endpoint | → `APPROVED`, `finishedAt = now` |
| `ReportRejectedFinal` | `bestDraftNumber, bestText, rejectionReason` | `drafts.size() == maxVerifyAttempts` AND last verdict is `REVISE`; OR compliance reviewer rejects; OR `defaultStepRecovery` failover | → `REJECTED_FINAL`, `finishedAt = now` |
| `EvalRecorded` | `draftNumber, verdict, score, wordCeilingExceeded, recordedAt` | `EvalSampler` per verified draft; workflow on terminal transition | (no status change; appended to `evalEvents[]` view-side projection) |

## Events (`QueryQueue`)

| Event | Payload |
|---|---|
| `QuerySubmitted` | `reportId, topic, sectorTag, wordCeiling, requestedBy, submittedAt` |

## View row

`ReportRow` mirrors `Report`. The `drafts` and `verifications` lists are bounded at `maxVerifyAttempts` (default 3) so the row stays small enough for SSE. The view's `TableUpdater` consumes every `ResearchEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllReports` returning the full list. Callers filter by `status` client-side.
