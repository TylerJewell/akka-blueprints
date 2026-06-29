# Architecture — sequential-cv-tailor

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph centres on two agents running in sequence. `CvEndpoint` accepts a `{candidateProfileId, jobPostingId}` POST, writes `CvRequestCreated` onto `CvRequestEntity`, and starts `CvPipelineWorkflow` keyed by `"cv-pipeline-" + requestId`. The workflow's first step (`generateStep`) emits `GenerationStarted`, then calls `CvGeneratorAgent` with `TaskDef.taskType(GENERATE_BASE_CV)` and the serialised `CandidateProfile` in the instruction context. Before each model call `PiiSanitizer` intercepts the payload, strips the five PII fields, and — if any were present — writes a `SanitizationApplied` event onto the entity for the audit trail.

Once `CvGeneratorAgent` returns a `BaseCv`, the workflow writes `BaseCvReady` and advances to `tailorStep`. This step calls `CvTailorAgent` with the `BaseCv` and `JobPosting` as instruction context — the raw `CandidateProfile` is never forwarded. `PiiSanitizer` fires again on the tailor call, though with a clean `BaseCv` payload it typically finds no PII fields to remove. Once `CvTailorAgent` returns a `TailoredCv`, the workflow writes `TailoredCvReady` and runs `evalStep`. `AlignmentScorer` checks keyword coverage deterministically — no LLM call — and writes `QualityScored`. `CvRequestView` projects every event into a read-model row; `CvEndpoint` serves it over REST and SSE.

The graph has exactly two LLM-calling components. `AlignmentScorer` is a deterministic rule-based scorer; that is what makes this a faithful **two-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The task boundary IS the data isolation contract. Between `generateStep` and `tailorStep`, the workflow writes `BaseCvReady` and reads the recorded `BaseCv` back from the entity to build the tailor task's instruction context. `CvTailorAgent` never sees the raw `CandidateProfile` — not because the profile is deleted, but because the workflow only forwards the typed result.
2. `PiiSanitizer` fires on every model call, not once at ingestion. The generator's task payload contains the raw profile (with PII); the sanitizer strips it before the model receives the payload. The tailor's task payload contains only `BaseCv` and `JobPosting` (no PII by construction), but the sanitizer still checks — a safety net for cases where a future field is added to `BaseCv` that inadvertently re-surfaces personal data.

## State machine

Seven states. The notable paths:

- The happy path walks `CREATED → GENERATING → GENERATED → TAILORING → TAILORED → SCORED`.
- Two failure transitions land in `FAILED`: a generator error during `GENERATING`, or a tailor error during `TAILORING`. A `FAILED` request's prior data is preserved on the entity — the UI shows the partial state (e.g., a `BaseCv` that was recorded before the tailor step failed).
- `SanitizationApplied` is a side-event recorded for audit; it does not transition status. The `sanitizationCount` field on `CvRequestRecord` is incremented on every such event.

There is no `SUBMITTED` or `ACCEPTED` state. The pipeline produces a tailored CV for recruiter review; the recruiter acts outside the system. The blueprint deliberately stops at `SCORED`.

## Entity model

`CvRequestEntity` is the source of truth. It emits eight event types: the two lifecycle starts (`GenerationStarted`, `TailoringStarted`), the two lifecycle completions (`BaseCvReady`, `TailoredCvReady`), the evaluation (`QualityScored`), the PII audit trail (`SanitizationApplied`), the failure (`RequestFailed`), and the initial creation (`CvRequestCreated`). `CvRequestView` projects every event into a row for the UI. `CvPipelineWorkflow` both reads (`getRequest`) and writes on the entity.

## Defence-in-depth governance flow

For any request that reaches `SCORED`, the candidate data passed through:

1. **PII sanitizer (before each model call)** — fires on every call to `CvGeneratorAgent` and `CvTailorAgent`. Fields `candidateName`, `email`, `phone`, `address`, `dateOfBirth` are replaced with `[REDACTED]` before the payload reaches the model. Each firing emits `SanitizationApplied` for the audit log.
2. **CvGeneratorAgent (1 task run)** — one model call, one structured `BaseCv` output. The typed result is the only thing the next stage receives.
3. **CvTailorAgent (1 task run)** — one model call, one structured `TailoredCv` output. Operates on `BaseCv` + `JobPosting` only; no raw profile.
4. **On-decision evaluator** — every emitted `TailoredCv` gets a 1–5 alignment score. Required-keyword coverage, preferred-keyword coverage, experience presence, and summary length are each worth one point.

Each step is independent. The sanitizer does not check alignment; the evaluator does not check PII presence. Removing one of them opens an explicit gap the other does not silently cover.
