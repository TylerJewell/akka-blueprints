# Architecture — composed-hiring-workflow

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`HiringEndpoint` is the entry point. A submitted application is logged as an `ApplicationSubmitted` event on `ApplicationQueue` (event-sourced for audit). `ApplicationRequestConsumer` subscribes to that queue, creates an `ApplicationEntity`, and starts a `HiringTeamWorkflow` keyed by the application id. That workflow is the top-level orchestrator: it runs the application through five stages, and two of those stages are themselves nested workflows.

The three desks under the pipeline each run a different internal coordination capability:

- **Screening desk — delegation (via `CandidateWorkflow`).** `screenStep` starts a nested `CandidateWorkflow`. Inside it, `ScreeningLead` plans dimensions, fans out one `Screener` instance per dimension (each writing a note into the workspace through `ApplicationTools`), then `ScreeningLead` synthesises the notes into a `ScreeningReport`. The nested workflow completes and returns the report to the outer pipeline.
- **CV improvement desk — a team feedback loop (via `CvImprovementLoop`).** `improveStep` starts a nested `CvImprovementLoop`. Inside it, `CvCoach` produces an improved CV draft and `CvCritic` scores it; the loop repeats up to three times. The nested workflow completes and returns the accepted draft.
- **Interview desk — moderation.** `interviewStep` runs one `Interviewer` instance per axis (`technical`, `behavioural`, `cultural`), each writing an `InterviewScore`; the deterministic `PanelRule` turns the scores into a `PanelVerdict`.

`StageEvalConsumer` subscribes to the application's stage events and records a non-blocking quality eval after each stage result. Two TimedActions sit alongside: `ApplicationSimulator` drips sample applications; `StaleDimensionMonitor` releases stranded screening dimension claims. `AppEndpoint` serves the embedded UI and the metadata the tabs read.

## Interaction sequence

The sequence diagram traces the happy path (J1): submit → open brief → CandidateWorkflow (screen and synthesise) → CvImprovementLoop (coach → pass critique) → interview panel (three axes) → PanelRule returns PROCEED → HiringManager drafts offer → offer extended. Two `Note over` blocks mark the asynchronous handoffs: the before-agent-response guardrail vets the `OfferLetter`, and the nested workflows run as sub-pipelines awaited by the outer workflow.

## State machine

`ApplicationEntity` is the spine of the pipeline. It moves `SUBMITTED → OPENED → SCREENING → SCREENED → CV_IMPROVING → CV_IMPROVED → INTERVIEWING`, and then either `SHORTLISTED → OFFER_PENDING → OFFER_EXTENDED` on a passing panel, or `DECLINED` on a DECLINE verdict. Two self-transitions matter: `recordOfferBlock` keeps the application `OFFER_PENDING` when the output guardrail refuses the letter (nothing is extended), and `recordPostHireReview` keeps the application `OFFER_EXTENDED` when a post-hire review lands — the compliance reviewer is on the loop, not gating it. A bounded `requestReassess` transition loops back from `INTERVIEWING` to `CV_IMPROVING` once before the pipeline forces a final verdict.

## Entity model

`ApplicationEntity` is the shared workspace and the source of truth for a hiring case; every stage writes one of eleven event types, and `ApplicationBoardView` is its read-side projection. `CvEntity` is the CV improvement tracking point — one per application, accumulating `CvDraft` iterations with an atomic `acceptDraft` that records the final chosen version; `CvBoardView` is its read-side projection. `ApplicationQueue` is the audit log of submissions. `StageEvalConsumer` reads application events to score stages; it writes its eval back through a command, not by mutating state directly.

## Concurrency & timeouts

- The top-level pipeline is a sequential orchestrator; nested workflows run as sub-tasks that the outer workflow awaits.
- `CandidateWorkflow` fans out screeners in parallel within its own step; `CvImprovementLoop` runs the coach-critic pair sequentially with an iteration counter bounding the loop.
- Per-step timeouts: `openStep` 60 s, `screenStep` 180 s (nests `CandidateWorkflow`, which fans out multiple screeners), `improveStep` 180 s (nests `CvImprovementLoop`), `interviewStep` 120 s (fans out three interviewer calls), `offerStep` 60 s. Inside `CvImprovementLoop`: `coachStep` 90 s, `criticStep` 60 s.
- The coach-critic loop runs at most three iterations, so the nested workflow always terminates.
- The revision loop in the outer pipeline runs at most once (one `REASSESS` re-assessment round), so the outer pipeline always terminates.
- `StaleDimensionMonitor` releases a screening dimension idle for more than three minutes so a failed screener does not strand `CandidateWorkflow`.
- `PanelRule` and `StageEvaluator` are deterministic pure functions (no LLM call), so the panel verdict and stage scores are reproducible.
