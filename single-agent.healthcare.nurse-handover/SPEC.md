# SPEC — nurse-handover

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** NurseHandover.
**One-line pitch:** A clinician submits an end-of-shift report and a ward checklist; one AI agent reads the report (passed as a task attachment, never as inline prompt text) and returns a structured `HandoverSummary` — per-patient status, outstanding tasks, risk flags — held open for explicit clinical sign-off before it is sealed.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the healthcare domain. One `HandoverAgent` (AutonomousAgent) carries the entire summarization decision; the surrounding components prepare its input, gate its output behind a human clinician, and audit the full lifecycle. Two governance mechanisms are wired around the agent:

- A **PHI sanitizer** runs inside a Consumer between the raw shift-report submission and the agent call — so the model never sees patient identifiers, MRNs, dates of birth, or insurance details.
- A **human-in-the-loop (HITL) sign-off gate** holds the handover in `AWAITING_SIGNOFF` state until a named clinician calls the sign-off endpoint. The handover cannot be `SIGNED_OFF` without that explicit action, and a sign-off by an unauthenticated or mismatched caller is rejected.

The blueprint shows that the single-agent pattern does not mean "ungoverned" — two independent controls sit on either side of the one decision-making LLM call, one before and one after.

## 3. User-facing flows

The user opens the App UI tab.

1. The clinician picks a **ward** from a dropdown (General Medicine, ICU, Paediatrics) — each ward has a pre-loaded checklist.
2. The clinician pastes an **end-of-shift report** into the textarea (or loads one of three seeded examples — a general-medicine 3-patient report, an ICU 2-patient report, a paediatrics 4-patient report).
3. The clinician fills in their **name** and clicks **Submit handover**. The UI POSTs to `/api/handovers` and receives a `handoverId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s it transitions to `SANITIZED` — the redacted report is visible in the card detail, with a list of PHI categories the sanitizer found.
5. Within ~10–30 s the `summarizeStep` completes. The card transitions to `SUMMARIZING` then `SUMMARY_READY`. The handover summary appears: a per-patient status table, a list of outstanding tasks ordered by urgency, and any risk flags.
6. The card enters `AWAITING_SIGNOFF`. The incoming clinician (who will take over the ward) sees a **Sign off** button. They click it; the UI PATCHes `/api/handovers/{id}/signoff` with their name.
7. The card transitions to `SIGNED_OFF`. The summary is now sealed — no further state changes. A `signedOffBy` and `signedOffAt` appear in the detail view.
8. The clinician can submit another handover; the live list shows the history.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `HandoverEndpoint` | `HttpEndpoint` | `/api/handovers/*` — submit, list, get, signoff, SSE; serves `/api/metadata/*`. | — | `HandoverEntity`, `HandoverView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `HandoverEntity` | `EventSourcedEntity` | Per-handover lifecycle: submitted → sanitized → summarizing → summary-ready → awaiting-signoff → signed-off / failed. Source of truth. | `HandoverEndpoint`, `ReportSanitizer`, `HandoverWorkflow` | `HandoverView` |
| `ReportSanitizer` | `Consumer` | Subscribes to `ReportSubmitted` events; redacts PHI; calls `HandoverEntity.attachSanitized`. | `HandoverEntity` events | `HandoverEntity` |
| `HandoverWorkflow` | `Workflow` | One workflow per handover. Steps: `awaitSanitizedStep` → `summarizeStep` → `awaitSignoffStep`. | started by `ReportSanitizer` once sanitized event lands | `HandoverAgent`, `HandoverEntity` |
| `HandoverAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the ward checklist in the task definition and the sanitized report as a task attachment; returns `HandoverSummary`. | invoked by `HandoverWorkflow` | returns summary |
| `HandoverView` | `View` | Read model: one row per handover for the UI. | `HandoverEntity` events | `HandoverEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ChecklistItem(String itemId, String description, String urgency) {}
// urgency: "ROUTINE" | "URGENT" | "CRITICAL"

record HandoverRequest(
    String handoverId,
    String ward,
    String shiftDate,
    String submittedBy,
    String rawReport,
    List<ChecklistItem> checklist,
    Instant submittedAt
) {}

record SanitizedReport(
    String redactedReport,
    List<String> phiCategoriesFound
) {}

record PatientStatus(
    String patientRef,        // e.g. "Patient-A" — de-identified label
    String currentCondition,
    String outstandingTasks,
    RiskLevel riskLevel
) {}
enum RiskLevel { LOW, MODERATE, HIGH, CRITICAL }

record HandoverSummary(
    List<PatientStatus> patients,
    List<String> outstandingTasks,    // ward-level tasks in urgency order
    List<String> riskFlags,           // items needing immediate attention
    String narrativeSummary,          // 2–4 sentences for the incoming clinician
    Instant summarizedAt
) {}

record SignoffRecord(
    String signedOffBy,
    Instant signedOffAt
) {}

record Handover(
    String handoverId,
    Optional<HandoverRequest> request,
    Optional<SanitizedReport> sanitized,
    Optional<HandoverSummary> summary,
    Optional<SignoffRecord> signoff,
    HandoverStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum HandoverStatus {
    SUBMITTED, SANITIZED, SUMMARIZING, SUMMARY_READY,
    AWAITING_SIGNOFF, SIGNED_OFF, FAILED
}
```

Events on `HandoverEntity`: `ReportSubmitted`, `ReportSanitized`, `SummarizationStarted`, `SummaryRecorded`, `SignoffRequested`, `HandoverSignedOff`, `HandoverFailed`.

Every nullable lifecycle field on the `Handover` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/handovers` — body `{ ward, shiftDate, submittedBy, rawReport, checklist: [ChecklistItem] }` → `{ handoverId }`.
- `GET /api/handovers` — list all handovers, newest-first.
- `GET /api/handovers/{id}` — one handover.
- `PATCH /api/handovers/{id}/signoff` — body `{ signedOffBy }` → `200` or `409` if already signed off.
- `GET /api/handovers/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: NurseHandover</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted handovers (status pill + ward badge + age + sign-off button when applicable) and a right pane with the selected handover's detail — submitted checklist, sanitized report preview, per-patient status table, outstanding tasks, risk flags, narrative summary, and sign-off status.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PHI sanitizer** (`phi`, applied inside `ReportSanitizer` Consumer): redacts patient MRNs, dates of birth, insurance identifiers, healthcare-provider names, facility-specific room numbers, and phone numbers from the raw shift report before any LLM call. Records which categories were found. The raw report is preserved on the entity for audit.
- **H1 — Clinical sign-off gate** (`hitl`, `application`): once `HandoverSummary` is ready, `HandoverWorkflow.awaitSignoffStep` waits for a `PATCH /api/handovers/{id}/signoff` call from a named clinician. The workflow polls `HandoverEntity.getHandover` every 5 s up to a configurable timeout (default 4 hours — long enough for a handover window). No transition to `SIGNED_OFF` happens without this call. A second sign-off attempt against an already-signed-off handover is rejected with `409 Conflict`.

## 9. Agent prompts

- `HandoverAgent` → `prompts/handover-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached shift report, walk every checklist item, produce a `PatientStatus` per de-identified patient reference, order outstanding tasks by urgency, and write a 2–4-sentence narrative for the incoming clinician.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Clinician submits a general-medicine seed report; within 30 s the handover summary appears with per-patient status rows and a narrative. Card enters `AWAITING_SIGNOFF`.
2. **J2** — Incoming clinician signs off via the UI button; card transitions to `SIGNED_OFF` with `signedOffBy` and `signedOffAt` populated. A second sign-off attempt returns `409`.
3. **J3** — A shift report containing `MRN 789-45-6123` and `DOB 1962-03-14` is submitted; the LLM call log shows only `[REDACTED-MRN]` and `[REDACTED-DOB]`; the entity's `request.rawReport` retains the raw text.
4. **J4** — A seeded ICU report is submitted; the summary lists the ICU-specific risk flags (ventilator weaning, vasopressor titration) and the urgency ordering puts `CRITICAL` tasks first.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named nurse-handover demonstrating the single-agent × healthcare cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-healthcare-nurse-handover. Java package io.akka.samples.nursehandover. Akka 3.6.0.
HTTP port 9170.

Components to wire (exactly):

- 1 AutonomousAgent HandoverAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/handover-agent.md>) and
  .capability(TaskAcceptance.of(SUMMARIZE_HANDOVER).maxIterationsPerTask(3)). The task receives
  the ward checklist as its instruction text and the sanitized shift report as a task ATTACHMENT
  (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the canonical
  call). Output: HandoverSummary{patients: List<PatientStatus>, outstandingTasks: List<String>,
  riskFlags: List<String>, narrativeSummary: String, summarizedAt: Instant}.

- 1 Workflow HandoverWorkflow per handoverId with three steps:
  * awaitSanitizedStep — polls HandoverEntity.getHandover every 1s; on
    handover.sanitized().isPresent() advances to summarizeStep.
    WorkflowSettings.stepTimeout 15s.
  * summarizeStep — emits SummarizationStarted, then calls
    componentClient.forAutonomousAgent(HandoverAgent.class, "handover-" + handoverId)
    .runSingleTask(TaskDef.instructions(formatChecklist(handover.request.checklist))
      .attachment("shift-report.txt",
                  handover.sanitized.redactedReport.getBytes())) — returns a taskId, then
    forTask(taskId).result(SUMMARIZE_HANDOVER) to fetch the summary. On success calls
    HandoverEntity.recordSummary(summary). WorkflowSettings.stepTimeout 60s with
    defaultStepRecovery maxRetries(2).failoverTo(HandoverWorkflow::error).
  * awaitSignoffStep — polls HandoverEntity.getHandover every 5s up to a step timeout of
    14400s (4 h). When handover.signoff().isPresent() is detected, calls
    HandoverEntity.completeSignoff() and ends the workflow. WorkflowSettings.stepTimeout
    14400s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity HandoverEntity (one per handoverId). State Handover{handoverId: String,
  request: Optional<HandoverRequest>, sanitized: Optional<SanitizedReport>,
  summary: Optional<HandoverSummary>, signoff: Optional<SignoffRecord>,
  status: HandoverStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  HandoverStatus enum: SUBMITTED, SANITIZED, SUMMARIZING, SUMMARY_READY,
  AWAITING_SIGNOFF, SIGNED_OFF, FAILED. Events: ReportSubmitted{request},
  ReportSanitized{sanitized}, SummarizationStarted{}, SummaryRecorded{summary},
  SignoffRequested{signedOffBy}, HandoverSignedOff{signoff}, HandoverFailed{reason}.
  Commands: submit, attachSanitized, markSummarizing, recordSummary, requestSignoff,
  completeSignoff, fail, getHandover. emptyState() returns Handover.initial("") with no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 Consumer ReportSanitizer subscribed to HandoverEntity events; on ReportSubmitted runs
  a regex+heuristic redaction pipeline (MRN patterns, DOB patterns, insurance-id patterns,
  phone numbers, provider name heuristic, room-number heuristic) over rawReport, computes
  the list of PHI categories found, builds SanitizedReport, then calls
  HandoverEntity.attachSanitized(sanitized). After attachSanitized lands, the same Consumer
  starts a HandoverWorkflow with id = "handover-" + handoverId.

- 1 View HandoverView with row type HandoverRow (mirrors Handover minus request.rawReport —
  the audit log keeps the raw; the view holds the sanitized form for the UI). Table updater
  consumes HandoverEntity events. ONE query getAllHandovers: SELECT * AS handovers FROM
  handover_view. No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2);
  caller filters client-side.

- 2 HttpEndpoints:
  * HandoverEndpoint at /api with POST /handovers (body
    {ward, shiftDate, submittedBy, rawReport,
     checklist: [{itemId, description, urgency}]};
    mints handoverId; calls HandoverEntity.submit; returns {handoverId}), GET /handovers
    (list from getAllHandovers, sorted newest-first), GET /handovers/{id} (one row),
    PATCH /handovers/{id}/signoff (body {signedOffBy}; calls HandoverEntity.requestSignoff;
    returns 200 or 409 if already SIGNED_OFF), GET /handovers/sse (Server-Sent Events
    forwarded from the view's stream-updates), and three /api/metadata/* endpoints serving
    the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- HandoverTasks.java declaring one Task<R> constant: SUMMARIZE_HANDOVER = Task.name("Summarize
  handover").description("Read the attached shift report and produce a HandoverSummary per
  checklist item").resultConformsTo(HandoverSummary.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records ChecklistItem, HandoverRequest, SanitizedReport, PatientStatus, RiskLevel,
  HandoverSummary, SignoffRecord, Handover, HandoverStatus.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9170 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/ward-checklists.jsonl with 3 seeded checklist sets:
  a 6-item General Medicine ward checklist, a 5-item ICU ward checklist, and a 7-item
  Paediatrics ward checklist.

- src/main/resources/sample-events/seed-reports.jsonl with 3 paired example shift reports:
  a synthetic General Medicine 3-patient report (900 words), a synthetic ICU 2-patient
  report (700 words), and a synthetic Paediatrics 4-patient report (1100 words). Each
  contains 2–3 plausible PHI strings so S1 has work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (S1, H1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.phi = true,
  phi_handled_by_sanitizer_before_llm = true, decisions.authority_level = recommend-only,
  oversight.human_in_loop = true, failure.failure_modes including
  "phi-leakage-via-llm", "missed-patient", "urgency-miscalibration",
  "unsigned-handover-acted-upon"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/handover-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: NurseHandover", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of handover cards; right = selected-handover detail with submitted checklist,
  sanitized report preview, per-patient status table, outstanding tasks, risk flags,
  narrative summary, and sign-off status).
  Browser title exactly: <title>Akka Sample: NurseHandover</title>. No subtitle on the
  Overview tab.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(handoverId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    summarize-handover.json — 6 HandoverSummary entries covering a range of patient
    counts (2–4 patients), urgency mixes, and risk-flag scenarios. Each entry has a
    non-empty narrativeSummary, a patients list with RiskLevel values, a non-empty
    outstandingTasks list, and a riskFlags list. Plus 1 deliberately MALFORMED entry
    (a PatientStatus with an empty patientRef) — exercises the mock path.
- A MockModelProvider.seedFor(handoverId) helper makes per-handover selection deterministic.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. HandoverAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion HandoverTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (awaitSanitizedStep 15s,
  summarizeStep 60s, awaitSignoffStep 14400s, error 5s).
- Lesson 6: every nullable lifecycle field on the Handover row record is Optional<T>.
- Lesson 7: HandoverTasks.java with SUMMARIZE_HANDOVER is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9170 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  and the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: exactly ONE AutonomousAgent (HandoverAgent). The HITL gate
  is a workflow polling step — not a second agent.
- The shift report is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated summarizeStep uses TaskDef.attachment(...).
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
