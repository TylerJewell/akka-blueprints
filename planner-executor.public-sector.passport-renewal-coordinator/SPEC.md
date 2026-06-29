# SPEC — passport-renewal-coordinator

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Passport Renewal Coordinator.
**One-line pitch:** Submit a renewal request; a Coordinator plans the verification steps on a renewal ledger, routes each step to a Document Reviewer, tokenizes PII before any LLM call, halts on missing documents, and gates submission to the issuing authority on caseworker approval.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern for a public-sector identity workflow. The `RenewalCoordinatorAgent` owns a **renewal ledger** (facts known, documents required, verification steps, current action) and a **review ledger** (each step's attempt count, verdict, blocker, reviewer notes). On each loop iteration the Coordinator reads both ledgers, picks the next action, and either continues, awaits the caseworker, escalates for missing docs, or completes.

The blueprint wires two governance mechanisms into that loop:

- a **PII sanitizer** that tokenizes passport numbers, national identity numbers, and date-of-birth values before they reach any LLM prompt — the Coordinator and DocumentReviewAgent see tokens only,
- a **human-in-the-loop approval gate** that pauses the workflow at the caseworker-wait step and resumes only when a caseworker has explicitly approved or rejected the submission via the API.

## 3. User-facing flows

The user opens the App UI tab and submits an application via the form.

1. The system creates an `Application` record in `SUBMITTED` and starts a `RenewalWorkflow`.
2. The Coordinator plans a `RenewalLedger { facts, requiredDocuments, steps, currentAction }` and emits `ApplicationPlanned`.
3. The workflow enters the review loop. Each iteration:
   - The Coordinator reads both ledgers and proposes a `ReviewDecision { action, documentId, rationale }`.
   - The **PII tokenizer** replaces raw identity fields (passport number, national ID, date of birth) with opaque tokens before passing context to `DocumentReviewAgent`.
   - `DocumentReviewAgent` checks the referenced document fixture and returns a `DocumentReviewResult { documentId, ok, notes, Optional<String> missingReason }`.
   - The workflow appends a `ReviewEntry { step, documentId, attempt, verdict, reviewerNotes, Optional<String> blocker, recordedAt }` to the review ledger.
   - If the reviewer reports a missing document, the workflow emits `DocumentsMissing` and the Application moves to `BLOCKED`.
4. When all required documents pass, the workflow emits `ReviewCompleted` and moves to `AWAITING_CASEWORKER`. A caseworker approval request is recorded on `CaseworkerControlEntity`.
5. **Human-in-the-loop gate:** the workflow pauses. A caseworker reviews the summary in the UI and calls `POST /api/applications/{id}/caseworker-approve` or `/caseworker-reject`. The workflow resumes on the resulting event.
6. On approval the workflow moves to `SUBMITTING`, calls the simulated agency endpoint, emits `AgencySubmitted`, and then `ApplicationCompleted`. The Application moves to `COMPLETED` with an `ApplicationAnswer { summary, referenceNumber }`.
7. On rejection the Application moves to `REJECTED` with the caseworker's stated reason.

A `RequestSimulator` (TimedAction) drips a sample application every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RenewalCoordinatorAgent` | `AutonomousAgent` | Plans the renewal, decides each review-loop iteration. Maintains the renewal ledger; reads the review ledger. Produces `ApplicationAnswer` on completion. | `RenewalWorkflow` | returns typed result to workflow |
| `DocumentReviewAgent` | `AutonomousAgent` | Examines document fixtures for completeness and validity. Returns `DocumentReviewResult`. | `RenewalWorkflow` | — |
| `RenewalWorkflow` | `Workflow` | Drives plan → check-halt → propose → pii-tokenize → document-review → record → await-caseworker → submit loop, plus blocked and reject branches. | `ApplicationEndpoint`, `ApplicationConsumer` | `ApplicationEntity` |
| `ApplicationEntity` | `EventSourcedEntity` | Holds the application lifecycle, renewal ledger, review ledger, caseworker decision, and final answer. | `RenewalWorkflow` | `ApplicationView` |
| `CaseworkerControlEntity` | `EventSourcedEntity` | Holds the per-application caseworker decision state. Keyed by `applicationId`. | `ApplicationEndpoint` (caseworker actions) | `RenewalWorkflow` (polls) |
| `ApplicationQueue` | `EventSourcedEntity` | Audit log of submitted applications. | `ApplicationEndpoint`, `RequestSimulator` | `ApplicationConsumer` |
| `ApplicationView` | `View` | List-of-applications read model for the UI. | `ApplicationEntity` events | `ApplicationEndpoint` |
| `ApplicationConsumer` | `Consumer` | Subscribes to `ApplicationQueue` events; starts a `RenewalWorkflow` per submission. | `ApplicationQueue` events | `RenewalWorkflow` |
| `RequestSimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/application-prompts.jsonl` and enqueues it. | scheduler | `ApplicationQueue` |
| `StaleApplicationMonitor` | `TimedAction` | Every 60 s, marks any application stuck in `REVIEWING_DOCS` past 10 minutes as `BLOCKED` with reason `"stale: no document progress"`. | scheduler | `ApplicationEntity` |
| `ApplicationEndpoint` | `HttpEndpoint` | `/api/applications/*` — submit, get, list, SSE, caseworker approve/reject. | — | `ApplicationView`, `ApplicationQueue`, `ApplicationEntity`, `CaseworkerControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record RenewalRequest(
    String applicantId,
    String rawPassportNumber,    // tokenized before LLM; never stored after tokenization
    String rawDateOfBirth,       // tokenized before LLM
    String rawNationalId,        // tokenized before LLM; may be empty
    String nationality,
    String expiryDate,
    String requestedBy
) {}

record PiiContext(
    String passportToken,        // opaque token substituting rawPassportNumber
    String dobToken,             // opaque token substituting rawDateOfBirth
    String nationalIdToken       // opaque token substituting rawNationalId; empty string if none
) {}

record RenewalLedger(
    List<String> facts,
    List<String> requiredDocuments,
    List<String> steps,
    Optional<ReviewDecision> currentAction
) {}

record ReviewDecision(
    String documentId,
    String checkType,
    String rationale
) {}

record DocumentReviewResult(
    String documentId,
    boolean ok,
    String notes,
    Optional<String> missingReason
) {}

record ReviewEntry(
    int attempt,
    String documentId,
    ReviewVerdict verdict,
    String reviewerNotes,
    Optional<String> blocker,
    Instant recordedAt
) {}

record ReviewLedger(List<ReviewEntry> entries) {}

record CaseworkerDecision(
    CaseworkerVerdict verdict,
    String decidedBy,
    String reason,
    Instant decidedAt
) {}

record ApplicationAnswer(
    String summary,
    String referenceNumber,
    Instant producedAt
) {}

record Application(
    String applicationId,
    String applicantId,
    String nationality,
    ApplicationStatus status,
    Optional<RenewalLedger> ledger,
    Optional<ReviewLedger> review,
    Optional<CaseworkerDecision> caseworkerDecision,
    Optional<ApplicationAnswer> answer,
    Optional<String> rejectionReason,
    Optional<String> blockReason,
    Instant submittedAt,
    Optional<Instant> finishedAt
) {}

enum ReviewVerdict { OK, MISSING, INVALID, NEEDS_RESUBMIT }
enum CaseworkerVerdict { APPROVED, REJECTED }
enum ApplicationStatus {
    SUBMITTED, PLANNING, REVIEWING_DOCS, AWAITING_CASEWORKER,
    SUBMITTING, COMPLETED, REJECTED, BLOCKED
}
```

### Events (`ApplicationEntity`)

`ApplicationCreated`, `ApplicationPlanned`, `DocumentReviewStarted`, `DocumentReviewRecorded`, `DocumentsMissing`, `ReviewCompleted`, `CaseworkerApprovalRequested`, `CaseworkerApproved`, `CaseworkerRejected`, `AgencySubmitted`, `ApplicationCompleted`, `ApplicationBlocked`.

### Events (`CaseworkerControlEntity`)

`CaseworkerDecisionRecorded { applicationId, verdict, decidedBy, reason, decidedAt }`.

### Events (`ApplicationQueue`)

`ApplicationSubmitted { applicationId, applicantId, nationality, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/applications` — body `{ applicantId, rawPassportNumber, rawDateOfBirth, rawNationalId?, nationality, expiryDate, requestedBy? }` → `202 { applicationId }`.
- `GET /api/applications` — list all applications. Optional `?status=...`.
- `GET /api/applications/{id}` — one application (full ledgers + caseworker decision + answer).
- `GET /api/applications/sse` — server-sent events stream of every application change.
- `POST /api/applications/{id}/caseworker-approve` — body `{ decidedBy, reason }` → `200`. Records caseworker approval; workflow resumes.
- `POST /api/applications/{id}/caseworker-reject` — body `{ decidedBy, reason }` → `200`. Records caseworker rejection; workflow ends in REJECTED.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Passport Renewal Coordinator"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a renewal application, caseworker approve/reject controls for applications awaiting review, live list of applications with status pills, expand-row to see the renewal ledger, review entries, caseworker decision, and final answer.

Browser title: `<title>Akka Sample: Passport Renewal Coordinator</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **P1 — PII sanitizer** (`sanitizer`, flavor `pii`): before any LLM call, `PiiTokenizer.tokenize(RenewalRequest)` replaces `rawPassportNumber`, `rawDateOfBirth`, and `rawNationalId` with opaque deterministic tokens (HMAC-SHA256 keyed by a per-service secret). The `PiiContext` record — tokens only — is what flows into agent prompts and the review ledger. Raw values never appear in events, view rows, logs, or LLM prompts.
- **H1 — caseworker approval gate** (`hitl`, flavor `application`): after all documents pass review, the workflow pauses at `caseworkerWaitStep`. A caseworker reads the application summary in the UI (rendered from tokenized data) and calls the approve or reject endpoint. `CaseworkerControlEntity` records the `CaseworkerDecisionRecorded` event; the workflow polls the entity and resumes. The agency submission endpoint is called only after `CaseworkerVerdict.APPROVED` is recorded. No auto-timeout bypasses this gate.

## 9. Agent prompts

- `RenewalCoordinatorAgent` → `prompts/renewal-coordinator.md`. Plans and decides each review step.
- `DocumentReviewAgent` → `prompts/document-reviewer.md`. Reviews one document fixture per call.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a complete renewal request. Application progresses `SUBMITTED → PLANNING → REVIEWING_DOCS → AWAITING_CASEWORKER → SUBMITTING → COMPLETED` within ~3 minutes after caseworker approval. UI reflects each transition via SSE. The expanded view shows a renewal ledger with a non-empty steps list, a review ledger with one entry per required document (all `OK`), a `CaseworkerDecision` with `verdict = APPROVED`, and a non-empty `ApplicationAnswer`.
2. **J2** — Submit a request that references a document not in the fixture set. The `DocumentReviewAgent` returns `ok=false, missingReason` set; the workflow emits `DocumentsMissing`; the application moves to `BLOCKED`. The `blockReason` names the specific missing document.
3. **J3** — Submit a complete request. While it is `AWAITING_CASEWORKER`, call the caseworker-reject endpoint with a stated reason. The application moves to `REJECTED`; `rejectionReason` is populated with the caseworker's reason; the agency submission endpoint is never called.
4. **J4** — Verify PII isolation. Submit any request carrying a real-format passport number. Confirm that no event payload, view row, LLM prompt log, or SSE message contains the raw passport number — only the opaque token appears downstream.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named passport-renewal-coordinator demonstrating the
planner-executor × public-sector cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact
planner-executor-public-sector-passport-renewal-coordinator. Java package
io.akka.samples.passportrenewalcoordinator. Akka 3.6.0. HTTP port 9232.

Components to wire (exactly):
- 2 AutonomousAgents:
  * RenewalCoordinatorAgent — definition() with three capabilities:
      capability(TaskAcceptance.of(PLAN_RENEWAL).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(DECIDE_NEXT).maxIterationsPerTask(4)) and
      capability(TaskAcceptance.of(COMPOSE_ANSWER).maxIterationsPerTask(2)).
    System prompt from prompts/renewal-coordinator.md.
    PLAN_RENEWAL returns RenewalLedger.
    DECIDE_NEXT returns a NextAction tagged union (Continue(ReviewDecision) |
    AwaitCaseworker | Reject(reason) | Complete(stub)).
    COMPOSE_ANSWER returns ApplicationAnswer.
  * DocumentReviewAgent — capability(TaskAcceptance.of(REVIEW_DOCUMENT)
      .maxIterationsPerTask(2)).
    Prompt from prompts/document-reviewer.md.
    Returns DocumentReviewResult.

- 1 Workflow RenewalWorkflow with steps:
  planStep -> [loop entry] checkCaseworkerStep -> proposeStep ->
  piiTokenizeStep -> documentReviewStep -> recordStep -> allDocsCheck ->
  [if complete: caseworkerWaitStep -> agencySubmitStep -> completeStep]
  [if missing: blockedStep] [if reject: rejectStep].
  Step timeouts:
    planStep ofSeconds(60), proposeStep ofSeconds(45),
    documentReviewStep ofSeconds(90), caseworkerWaitStep ofSeconds(0)
    (no timeout — gate waits indefinitely until caseworker acts),
    agencySubmitStep ofSeconds(30), completeStep ofSeconds(60).
  defaultStepRecovery(maxRetries(2).failoverTo(RenewalWorkflow::error)).
  checkCaseworkerStep polls CaseworkerControlEntity.get(applicationId);
  if a CaseworkerDecision is already recorded it routes to either
  agencySubmitStep (APPROVED) or rejectStep (REJECTED); otherwise
  it transitions to proposeStep.
  piiTokenizeStep applies PiiTokenizer.tokenize(renewalRequest) to produce
  a PiiContext; the raw fields are not forwarded to any subsequent step.
  documentReviewStep calls forAutonomousAgent(DocumentReviewAgent.class,
  REVIEW_DOCUMENT) with the ReviewDecision and PiiContext (no raw PII).
  caseworkerWaitStep emits CaseworkerApprovalRequested on ApplicationEntity,
  then transitions to a polling sub-loop reading CaseworkerControlEntity
  every 5 s until a decision arrives.

- 1 EventSourcedEntity ApplicationEntity holding Application state.
  Commands: createApplication, recordPlan, recordDocReview, recordMissing,
  completeReview, requestCaseworker, recordCaseworkerDecision,
  submitToAgency, completeApplication, blockApplication, getApplication.
  Events as listed in SPEC §5.

- 1 EventSourcedEntity CaseworkerControlEntity keyed by applicationId.
  State CaseworkerState { Optional<CaseworkerDecision> decision }.
  Commands: recordDecision, get. Event: CaseworkerDecisionRecorded.

- 1 EventSourcedEntity ApplicationQueue with command enqueueApplication
  emitting ApplicationSubmitted.

- 1 View ApplicationView with row type ApplicationRow (mirror of Application
  minus heavy review payload — truncate to last 3 review entries plus
  counts; the UI fetches the full application by id on click). ONE query
  getAllApplications SELECT * AS applications FROM application_view. No
  WHERE status filter — caller filters client-side (Lesson 2).

- 1 Consumer ApplicationConsumer subscribed to ApplicationQueue events; on
  ApplicationSubmitted starts a RenewalWorkflow with applicationId as the
  workflow id.

- 2 TimedActions:
  * RequestSimulator — every 90 s, reads next line from
    src/main/resources/sample-events/application-prompts.jsonl and calls
    ApplicationQueue.enqueueApplication.
  * StaleApplicationMonitor — every 60 s, queries ApplicationView
    .getAllApplications, filters REVIEWING_DOCS applications whose
    submittedAt is older than 10 minutes, calls
    ApplicationEntity.blockApplication with reason
    "stale: no document progress after 10m".

- 2 HttpEndpoints:
  * ApplicationEndpoint at /api with POST /applications,
    GET /applications (client-side filtered from getAllApplications),
    GET /applications/{id}, GET /applications/sse,
    POST /applications/{id}/caseworker-approve,
    POST /applications/{id}/caseworker-reject, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- CoordinatorTasks.java declaring three Task<R> constants: PLAN_RENEWAL
  (resultConformsTo RenewalLedger), DECIDE_NEXT (resultConformsTo NextAction),
  COMPOSE_ANSWER (resultConformsTo ApplicationAnswer).
- ReviewerTasks.java declaring one Task<R> constant: REVIEW_DOCUMENT
  (resultConformsTo DocumentReviewResult).
- NextAction sealed interface with permits:
  Continue(ReviewDecision), AwaitCaseworker, Reject(String reason),
  Complete(ApplicationAnswer stub).
- application/PiiTokenizer.java — deterministic HMAC-SHA256 tokenizer.
  tokenize(RenewalRequest) returns PiiContext. Key loaded from
  application.conf pii.tokenizer-key (a random base64 string generated
  at scaffold time; stored in conf, not in version control). Each raw
  field is tokenized as HMAC-SHA256(key, fieldValue) truncated to 12 hex
  chars, prefixed by type: ppt-<hex>, dob-<hex>, nid-<hex>. Empty
  nationalId produces empty token. The same input always produces the
  same token within a service instance (deterministic for replay).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9232 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/application-prompts.jsonl with 8 canned
  renewal applications (varying nationality, document sets, urgency).
- src/main/resources/sample-data/documents/*.md — 8 short document
  fixture files (photo-id.md, passport-copy.md, supporting-letter.md,
  proof-of-address.md, travel-itinerary.md, birth-certificate.md,
  name-change-deed.md, prior-passport-copy.md). Each fixture contains
  plausible but fictional details. One fixture (passport-copy.md) contains
  a raw-format passport number `X12345678` to exercise the PII isolation
  test (the agent must never see this literal value — only its token).
- src/main/resources/sample-data/agency-responses.jsonl — 6 canned agency
  endpoint responses: 4 success (with a reference number), 2 failure (with
  a retry-after field).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the project-root files for the metadata endpoint to serve
  from classpath).
- eval-matrix.yaml at the project root with 2 controls (P1, H1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose, data,
  decisions, failure, oversight, operations, and compliance.capabilities;
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/renewal-coordinator.md, prompts/document-reviewer.md loaded at
  agent startup as system prompts.
- README.md at the project root as described above.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows), App UI (form + caseworker approve/reject controls + live list
  with status pills and expand-on-click for ledgers and decision).
  Browser title exactly:
  <title>Akka Sample: Passport Renewal Coordinator</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and
  proceed silently.
- If none is set, ask the user how to source the key, offering five
  options via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with per-agent dispatch on agent class name
  and Task<R> id. Each branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json and picks one entry
  pseudo-randomly per call.
- Per-agent mock-response shapes for THIS blueprint:
    renewal-coordinator.json — three lists keyed by task id:
      "PLAN_RENEWAL" → 4 RenewalLedger entries (facts, requiredDocuments
      covering photo-id/passport-copy/supporting-letter, steps).
      "DECIDE_NEXT" → 6 NextAction entries cycling through Continue
      (covering each required document in sequence), then AwaitCaseworker,
      to exercise the full happy path.
      "COMPOSE_ANSWER" → 3 ApplicationAnswer entries with 60–100 word
      summaries and plausible reference numbers (PRN-2026-XXXX).
    document-reviewer.json — 8 DocumentReviewResult entries, one per
      fixture file. Seven are ok=true; one (for "name-change-deed.md")
      is ok=false, missingReason="Name change deed not present in fixture
      set" to exercise J2.
- A MockModelProvider.seedFor(applicationId) helper makes selection
  deterministic per application id.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent — both agents extend AutonomousAgent.
- Lesson 4: WorkflowSettings.stepTimeout set on planStep, proposeStep,
  documentReviewStep, agencySubmitStep, completeStep. caseworkerWaitStep
  has no timeout by design (indefinite human gate).
- Lesson 6: Optional<T> for every nullable field on ApplicationRow and
  on the Application entity state.
- Lesson 7: CoordinatorTasks.java and ReviewerTasks.java declare every
  Task<R> constant.
- Lesson 8: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: /akka:build.
- Lesson 10: HTTP port 9232 in application.conf.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no competitor brand names.
- Lesson 24: mermaid CSS overrides and themeVariables present.
- Lesson 25: five-option key-sourcing flow; no key value written to disk.
- Lesson 26: tab switching by data-tab / data-panel; no zombie panels.
- The Overview tab's Try-it card shows just "/akka:build".
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing flow in Section 11 covers this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
