# Data model — composed-hiring-workflow

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ApplicationBrief` | `applicationId` | `String` | no | Id assigned at submission. |
| | `candidateName` | `String` | no | The candidate's full name. |
| | `jobRole` | `String` | no | The role the candidate is applying for. |
| | `cvText` | `String` | no | The candidate's original CV text. |
| `HiringBrief` | `roleSummary` | `String` | no | One to two sentences describing the role. |
| | `requiredDimensions` | `List<String>` | no | Screening dimensions. |
| | `targetCompetencies` | `List<String>` | no | Interview competency labels. |
| `ScreeningPlan` | `dimensions` | `List<String>` | no | Dimensions, one per screener instance. |
| `ScreeningNote` | `dimension` | `String` | no | The dimension this note evaluates. |
| | `outcome` | `ScreeningOutcome` | no | `PASS`, `FAIL`, or `PENDING`. |
| | `comments` | `String` | no | Evidence-based explanation. |
| `ScreeningReport` | `summary` | `String` | no | One-paragraph synthesis of all notes. |
| | `notes` | `List<ScreeningNote>` | no | All screener notes. |
| | `overallOutcome` | `ScreeningOutcome` | no | `PASS` if threshold met; `FAIL` otherwise. |
| `CvDraft` | `candidateName` | `String` | no | Candidate name carried through the loop. |
| | `jobRole` | `String` | no | Target role. |
| | `revisedCvText` | `String` | no | Full revised CV text (non-empty). |
| | `iteration` | `int` | no | Which iteration produced this draft (1-indexed). |
| | `changesApplied` | `List<String>` | no | Short phrases naming each change made. |
| `CritiqueNote` | `iteration` | `int` | no | Iteration being critiqued. |
| | `outcome` | `CritiqueOutcome` | no | `PASS` or `REVISE`. |
| | `fitScore` | `int` | no | 0–100 fit score. |
| | `clarityScore` | `int` | no | 0–100 clarity score. |
| | `completenessScore` | `int` | no | 0–100 completeness score. |
| | `suggestions` | `List<String>` | no | Actionable suggestions (empty on PASS). |
| `InterviewScore` | `axis` | `String` | no | The competency axis (`technical`/`behavioural`/`cultural`). |
| | `outcome` | `InterviewOutcome` | no | `PROCEED`, `DECLINE`, or `REASSESS`. |
| | `score` | `int` | no | 0–100 score on this axis. |
| | `comments` | `String` | no | Evidence-based explanation. |
| `PanelVerdict` | `outcome` | `InterviewOutcome` | no | `PROCEED` only when all axes pass. |
| | `scores` | `List<InterviewScore>` | no | All interviewer scores. |
| | `reassessCompetencies` | `List<String>` | no | Axes flagged REASSESS (empty on PROCEED/DECLINE). |
| `OfferLetter` | `candidateName` | `String` | no | Candidate name on the letter. |
| | `jobRole` | `String` | no | Role being offered. |
| | `offerText` | `String` | no | Full offer body text. |
| | `offerReference` | `String` | no | Non-empty reference code (gated by G2). |
| | `draftedAt` | `Instant` | no | When the hiring manager assembled it. |
| `StageEval` | `stage` | `String` | no | `screening` / `cv_improvement` / `interview`. |
| | `score` | `int` | no | Quality score 0-100 from `StageEvaluator`. |
| | `flags` | `List<String>` | no | Issues found (may be empty). |
| | `evaluatedAt` | `Instant` | no | When the eval ran. |
| `PostHireReview` | `reviewedBy` | `String` | no | Compliance officer id. |
| | `outcome` | `PostHireOutcome` | no | `CLEARED` or `FLAGGED`. |
| | `comments` | `String` | no | Post-hire review notes. |
| | `reviewedAt` | `Instant` | no | When recorded. |

## Entity state — `Application` (`ApplicationEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `applicationId` | `String` | no | Unique id; also the `HiringTeamWorkflow` id. |
| `candidateName` | `String` | no | Candidate's full name. |
| `jobRole` | `String` | no | Role being applied for. |
| `cvText` | `String` | no | Original CV text. |
| `status` | `ApplicationStatus` | no | See enum. |
| `hiringBrief` | `Optional<HiringBrief>` | yes | Populated on `BriefOpened`. |
| `screeningReport` | `Optional<ScreeningReport>` | yes | Populated on `ScreeningCompleted`. |
| `acceptedCvDraft` | `Optional<CvDraft>` | yes | Populated on `CvImproved`. |
| `dimensionIds` | `List<String>` | no | Dimension slot ids created from the plan (may be empty early). |
| `panelVerdict` | `Optional<PanelVerdict>` | yes | Populated on `InterviewCompleted`. |
| `offerLetter` | `Optional<OfferLetter>` | yes | Populated on `OfferExtended`. |
| `offerReference` | `Optional<String>` | yes | Generated reference on offer. |
| `offerExtendedAt` | `Optional<Instant>` | yes | When the offer was extended. |
| `stageEvals` | `List<StageEval>` | no | Eval results from `StageEvalConsumer` (may be empty). |
| `postHireReview` | `Optional<PostHireReview>` | yes | Populated on `PostHireReviewRecorded`. |
| `reassessCount` | `int` | no | Number of re-assessment rounds taken (0 or 1). |
| `createdAt` | `Instant` | no | When `ApplicationCreated` emitted. |

## Entity state — `CvIteration` (`CvEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `applicationId` | `String` | no | Owning application. |
| `originalCvText` | `String` | no | The unmodified CV submitted with the application. |
| `drafts` | `List<CvDraft>` | no | All drafts produced by the coach (in order). |
| `acceptedDraft` | `Optional<CvDraft>` | yes | The draft the loop accepted. |
| `latestCritique` | `Optional<CritiqueNote>` | yes | The most recent critic's note. |
| `iterationCount` | `int` | no | Total iterations run so far. |
| `createdAt` | `Instant` | no | When `CvCreated` emitted. |

## Enums

`ApplicationStatus`: `SUBMITTED`, `OPENED`, `SCREENING`, `SCREENED`, `CV_IMPROVING`, `CV_IMPROVED`, `INTERVIEWING`, `SHORTLISTED`, `OFFER_PENDING`, `OFFER_EXTENDED`, `DECLINED`.
`ScreeningOutcome`: `PASS`, `FAIL`, `PENDING`.
`CritiqueOutcome`: `PASS`, `REVISE`.
`InterviewOutcome`: `PROCEED`, `DECLINE`, `REASSESS`.
`PostHireOutcome`: `CLEARED`, `FLAGGED`.

## Events — `ApplicationEntity`

| Event | Payload | Transition |
|---|---|---|
| `ApplicationCreated` | `applicationId, candidateName, jobRole, cvText, createdAt` | → SUBMITTED |
| `BriefOpened` | `hiringBrief` | SUBMITTED → OPENED |
| `ScreeningCompleted` | `screeningReport` | SCREENING → SCREENED |
| `CvImproved` | `acceptedCvDraft` | CV_IMPROVING → CV_IMPROVED |
| `InterviewCompleted` | `panelVerdict` | INTERVIEWING → SHORTLISTED (on PROCEED) or DECLINED (on DECLINE) |
| `ReassessRequested` | `reassessCompetencies, reassessCount` | INTERVIEWING → CV_IMPROVING (one bounded round) |
| `OfferDrafted` | `offerLetter` | SHORTLISTED → OFFER_PENDING |
| `OfferExtended` | `offerReference, offerExtendedAt` | OFFER_PENDING → OFFER_EXTENDED |
| `OfferBlocked` | `reason` | (no status change; leaves application OFFER_PENDING) |
| `StageEvaluated` | `stageEval` | (no status change; appends to `stageEvals`) |
| `PostHireReviewRecorded` | `postHireReview` | (no status change; OFFER_EXTENDED only) |

Note: the implementation may emit an explicit `ScreeningStarted` event for the `OPENED → SCREENING` edge or fold it into the first screener write — prefer whichever keeps the projection and the App UI status consistent.

## Events — `CvEntity`

| Event | Payload | Transition |
|---|---|---|
| `CvCreated` | `applicationId, originalCvText, createdAt` | → initial state |
| `DraftAdded` | `cvDraft` | (appends to `drafts`; increments `iterationCount`) |
| `DraftAccepted` | `cvDraft` | (sets `acceptedDraft`) |

## Events — `ApplicationQueue`

| Event | Payload |
|---|---|
| `ApplicationSubmitted` | `applicationId, candidateName, jobRole, submittedAt` |

## View rows

`ApplicationRow` (row of `ApplicationBoardView`) mirrors `Application` but drops the heavy `OfferLetter.offerText` and the `acceptedCvDraft.revisedCvText` — it keeps the brief role summary, the screening report summary and outcome, the accepted draft iteration count, the verdict outcome and `reassessCompetencies`, the `stageEvals`, the `offerReference`, and the `postHireReview`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

`CvRow` (row of `CvBoardView`) mirrors `CvIteration` but drops all full `revisedCvText` fields — it keeps `iterationCount` and a `draftPresent` boolean so the board and the improvement loop stay small. Every nullable lifecycle field is `Optional<T>`.

Both views expose one unfiltered query (`getAllApplications` / `getAllCvIterations`) plus a streaming query for the SSE endpoints. Status filtering happens client-side because the views cannot auto-index the enum status column (Lesson 2).
