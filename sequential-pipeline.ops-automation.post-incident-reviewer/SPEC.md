# SPEC — post-incident-reviewer

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Post-Incident Review Agent.
**One-line pitch:** An operator submits an incident ID; one `PIRAgent` walks it through three task phases — **GATHER** raw evidence, **ASSESS** impact and root cause, **DRAFT** a structured review — with each phase gated on the prior phase's recorded output, and the final draft reviewed by a `before-agent-response` guardrail and signed off by the incident owner before the review is marked complete.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in an ops-automation domain. One `PIRAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the GATHER task's typed output becomes the ASSESS task's instruction context; the ASSESS task's typed output becomes the DRAFT task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Two governance mechanisms are wired around the pipeline:

- A **`before-agent-response` guardrail** fires after the final DRAFT task and before the `ReviewDrafted` event is written to the entity. `PIRGuardrail` checks that the draft contains a non-empty executive summary, at least one action item with an owner and due date, and that every timeline entry cited in the review body is present in the recorded `EvidenceLog`. A draft missing any of these is rejected with a structured reason; the agent receives the rejection and retries within its 4-iteration budget. The same guardrail also ensures the impact classification (P1–P4) is consistent with the blast-radius evidence in the `ImpactAssessment`.
- A **human-in-the-loop sign-off** at the application layer. After `ReviewDrafted` lands, the workflow enters `signoffStep`, which calls `PIRSignoffNotifier` to alert the incident owner and then pauses. The workflow resumes only when the incident owner calls `POST /api/pir/{id}/signoff` with `{ approved: true/false, comments: ... }`. An approval transitions to `COMPLETE`; a rejection transitions to `REJECTED`. Both outcomes are recorded on the entity and reflected in the UI.

The blueprint shows that a sequential pipeline is not just a chain of LLM calls — the task-boundary handoffs enforce the dependency contract, the guardrail enforces output quality, and the HITL gate enforces accountability.

## 3. User-facing flows

The operator opens the App UI tab.

1. The operator enters an **incident ID** into the input (or picks one of three seeded incidents — `INC-2026-0741`, `INC-2026-0829`, `INC-2026-0903`).
2. The operator clicks **Start PIR**. The UI POSTs to `/api/pir` and receives a `pirId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `GATHERING` — the workflow has started `gatherStep` and the agent has been handed the GATHER task.
4. Within ~10–20 s the card reaches `ASSESSED` — the typed `EvidenceLog` and `ImpactAssessment` are visible in the card detail. The agent's GATHER task returned; the workflow recorded `EvidenceGathered` and ran the ASSESS task, which recorded `ImpactAssessed`.
5. Within ~10–20 s more the card reaches `DRAFTED`. The full draft `PostIncidentReview` is visible — executive summary, impact classification, timeline, root-cause analysis, and action items list.
6. The card shows `AWAITING_SIGNOFF`. A sign-off banner appears with the assignee's name and an approve/reject button pair in the right pane.
7. The operator (acting as incident owner in this demo) clicks **Approve**. The UI POSTs `{ approved: true }` to `/api/pir/{pirId}/signoff`. Within ~1 s the card transitions to `COMPLETE`.
8. The operator can submit another incident; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PIREndpoint` | `HttpEndpoint` | `/api/pir/*` — submit, list, get, signoff, SSE; serves `/api/metadata/*`. | — | `PIREntity`, `PIRView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `PIREntity` | `EventSourcedEntity` | Per-review lifecycle: created → gathering → gathered → assessing → assessed → drafting → drafted → awaiting_signoff → complete / rejected. Source of truth. | `PIREndpoint`, `PIRWorkflow` | `PIRView` |
| `PIRWorkflow` | `Workflow` | One workflow per pirId. Steps: `gatherStep` → `assessStep` → `draftStep` → `signoffStep`. Each agent step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. `signoffStep` pauses until the incident owner signals. | started by `PIREndpoint` after `CREATED` | `PIRAgent`, `PIREntity`, `PIRSignoffNotifier` |
| `PIRAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `PIRTasks.java`: `GATHER_EVIDENCE` → `EvidenceLog`, `ASSESS_IMPACT` → `ImpactAssessment`, `DRAFT_REVIEW` → `PostIncidentReview`. Each task is registered with the phase-appropriate function tools. The `before-agent-response` guardrail (`PIRGuardrail`) is registered on the agent and fires on the DRAFT task's response. | invoked by `PIRWorkflow` | returns typed results |
| `GatherTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `fetchIncidentRecord(incidentId)` and `fetchTimelineEvents(incidentId)`. Reads from `src/main/resources/sample-data/incidents/*.json` for deterministic offline output. | called from GATHER task | returns `IncidentRecord`, `List<TimelineEvent>` |
| `AssessTools` | function-tools class | Implements `classifyImpact(evidenceLog)` and `identifyRootCause(evidenceLog)`. Pure in-memory transformations. | called from ASSESS task | returns `ImpactClassification`, `RootCause` |
| `DraftTools` | function-tools class | Implements `composeExecutiveSummary(assessment)` and `buildActionItems(assessment, rootCause)`. | called from DRAFT task | returns `String` (summary), `List<ActionItem>` |
| `PIRGuardrail` | `before-agent-response` guardrail (registered on `PIRAgent`) | Checks the draft response from the DRAFT task: non-empty executive summary, ≥ 1 action item with owner+due-date, all cited timeline references present in `EvidenceLog`, impact classification consistent with blast radius. Rejects non-compliant drafts with a structured reason. | DRAFT task response | accept / structured-reject |
| `PIRSignoffNotifier` | plain class (no Akka primitive) | Called from `signoffStep`. Sends a sign-off request notification (in-process webhook stub). Records the assignee's email and the review URL on the entity event. | called from `signoffStep` | side-effect only |
| `PIRView` | `View` | Read model: one row per review for the UI. | `PIREntity` events | `PIREndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record IncidentRecord(
    String incidentId,
    String title,
    String severity,       // P1 | P2 | P3 | P4
    String reportedBy,
    Instant detectedAt,
    Instant resolvedAt
) {}

record TimelineEvent(
    String eventId,
    Instant occurredAt,
    String actor,
    String description
) {}

record EvidenceLog(
    IncidentRecord incident,
    List<TimelineEvent> timeline,
    Instant gatheredAt
) {}

record ImpactClassification(
    String severity,       // P1 | P2 | P3 | P4
    String affectedSystems,
    int usersAffected,
    Duration outageWindow
) {}

record RootCause(
    String rootCauseId,
    String summary,
    List<String> contributingFactors
) {}

record ImpactAssessment(
    ImpactClassification classification,
    RootCause rootCause,
    Instant assessedAt
) {}

record ActionItem(
    String actionId,
    String description,
    String owner,
    LocalDate dueDate,
    String priority       // HIGH | MEDIUM | LOW
) {}

record PostIncidentReview(
    String pirId,
    String executiveSummary,
    ImpactClassification impactClassification,
    List<TimelineEvent> timeline,
    RootCause rootCause,
    List<ActionItem> actionItems,
    Instant draftedAt
) {}

record SignoffDecision(
    boolean approved,
    Optional<String> comments,
    String decidedBy,
    Instant decidedAt
) {}

record PIRRecord(
    String pirId,
    Optional<String> incidentId,
    Optional<EvidenceLog> evidenceLog,
    Optional<ImpactAssessment> impactAssessment,
    Optional<PostIncidentReview> review,
    Optional<SignoffDecision> signoffDecision,
    PIRStatus status,
    Instant createdAt,
    Optional<Instant> completedAt
) {}

enum PIRStatus {
    CREATED, GATHERING, GATHERED, ASSESSING, ASSESSED,
    DRAFTING, DRAFTED, AWAITING_SIGNOFF, COMPLETE, REJECTED, FAILED
}
```

Events on `PIREntity`: `PIRCreated`, `GatherStarted`, `EvidenceGathered`, `AssessStarted`, `ImpactAssessed`, `DraftStarted`, `ReviewDrafted`, `GuardrailRejected`, `SignoffRequested`, `SignoffRecorded`, `ReviewComplete`, `ReviewRejected`, `PIRFailed`.

Every nullable lifecycle field on the `PIRRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/pir` — body `{ incidentId }` → `{ pirId }`.
- `GET /api/pir` — list all reviews, newest-first.
- `GET /api/pir/{id}` — one review.
- `POST /api/pir/{id}/signoff` — body `{ approved, comments }` → `{ pirId, status }`.
- `GET /api/pir/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Post-Incident Review Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of reviews (status pill + incident ID + age) and a right pane with the selected review's detail — incident record, timeline table, impact assessment, full review document, sign-off panel, and a guardrail-rejection log strip if any draft-quality rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-agent-response` guardrail (draft-quality gate)**: `PIRGuardrail` is registered on `PIRAgent` and runs after the DRAFT task returns its response but before the `ReviewDrafted` event is written to the entity. It applies four quality checks: (1) executive summary is non-empty and ≥ 50 characters; (2) at least one `ActionItem` with a non-null `owner` and non-null `dueDate` is present; (3) every `TimelineEvent.eventId` cited in the review body is present in the `EvidenceLog.timeline`; (4) the `PostIncidentReview.impactClassification.severity` matches the severity recorded in `EvidenceLog.incident.severity`. On reject, the guardrail returns a structured `draft-quality-violation` reason to the agent loop and writes a `GuardrailRejected{check, reason}` event for visibility. The agent retries within its 4-iteration budget.
- **H1 — HITL sign-off (application gate)**: after `ReviewDrafted` lands and the guardrail has accepted the draft, `PIRWorkflow.signoffStep` calls `PIRSignoffNotifier` to record the sign-off request and then pauses. The workflow resumes when the incident owner calls `POST /api/pir/{id}/signoff`. An `approved: true` decision writes `SignoffRecorded` and then `ReviewComplete`; an `approved: false` decision writes `SignoffRecorded` and then `ReviewRejected`. The sign-off endpoint is the only way to advance past `AWAITING_SIGNOFF`; the workflow does not time out on this step — accountability requires a human decision.

## 9. Agent prompts

- `PIRAgent` → `prompts/pir-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output. The DRAFT task is additionally instructed to produce a guardrail-compliant response: every action item must name an owner, every timeline reference must come from the evidence log passed in its instructions.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Operator submits the seeded incident `INC-2026-0741`; within 60 s the review reaches `AWAITING_SIGNOFF` with a non-empty executive summary, ≥ 2 action items, and a timeline with ≥ 3 entries.
2. **J2** — The agent's DRAFT task response is missing action items (mock LLM path). `PIRGuardrail` rejects the response; a `GuardrailRejected` event lands; the agent retries with a valid draft; the review eventually reaches `AWAITING_SIGNOFF`. The guardrail-rejection strip in the UI shows the one rejected response.
3. **J3** — The incident owner POSTs `{ approved: true }` to the sign-off endpoint; the workflow resumes; the entity transitions to `COMPLETE`; the UI card shows a green `COMPLETE` pill.
4. **J4** — The incident owner POSTs `{ approved: false, comments: "missing blast-radius scope" }`; the entity transitions to `REJECTED`; the UI card flags the review with the rejection comment.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named post-incident-reviewer demonstrating the sequential-pipeline x
ops-automation cell. Runs out of the box (no external services). Maven group io.akka.samples.
Maven artifact sequential-pipeline-ops-automation-post-incident-reviewer.
Java package io.akka.samples.postincidentreviewagent. Akka 3.6.0. HTTP port 9423.

Components to wire (exactly):

- 1 AutonomousAgent PIRAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/pir-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  GATHER, ASSESS, and DRAFT tool sets are ALL registered on the agent; phase discipline is the
  agent's responsibility per its system prompt. The before-agent-response guardrail
  (PIRGuardrail) is registered on the agent via the agent's guardrail-configuration block,
  bound to respond-hook (fires on DRAFT task response only — the guardrail checks the current
  task name in the response context and passes GATHER/ASSESS responses through immediately).
  On guardrail rejection the agent loop retries within its 4-iteration budget.

- 1 Workflow PIRWorkflow per pirId with four steps:
  * gatherStep — emits GatherStarted on the entity, then calls componentClient
    .forAutonomousAgent(PIRAgent.class, "agent-" + pirId).runSingleTask(
      TaskDef.instructions("IncidentId: " + incidentId + "\nPhase: GATHER\nFetch the
      incident record and all timeline events for this incident.")
        .metadata("pirId", pirId)
        .metadata("phase", "GATHER")
        .taskType(PIRTasks.GATHER_EVIDENCE)
    ). Reads forTask(taskId).result(GATHER_EVIDENCE) to get EvidenceLog. Writes
    PIREntity.recordEvidence(evidenceLog). WorkflowSettings.stepTimeout 60s.
  * assessStep — emits AssessStarted, then runSingleTask with TaskDef.instructions
    (formatAssessContext(evidenceLog, incidentId)) and metadata.phase = "ASSESS", taskType
    ASSESS_IMPACT. Writes PIREntity.recordAssessment(impactAssessment). stepTimeout 60s.
  * draftStep — emits DraftStarted, then runSingleTask with TaskDef.instructions
    (formatDraftContext(impactAssessment, evidenceLog, incidentId)) and metadata.phase = "DRAFT",
    taskType DRAFT_REVIEW. The PIRGuardrail fires on this task's response before
    PIREntity.recordReview(review) is called. Writes PIREntity.recordReview(review).
    stepTimeout 90s (extra budget for guardrail retries).
  * signoffStep — calls PIRSignoffNotifier.notify(pirId, assigneeEmail) and then calls
    PIREntity.requestSignoff(assigneeEmail). Returns a WorkflowState.waitForSignal (the step
    does NOT complete until resumeSignoff() is called externally). No stepTimeout on this step
    — human accountability requires a human decision; the workflow waits indefinitely.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(PIRWorkflow::error). The error step writes PIRFailed and ends.

- 1 EventSourcedEntity PIREntity (one per pirId). State PIRRecord{pirId,
  incidentId: Optional<String>, evidenceLog: Optional<EvidenceLog>,
  impactAssessment: Optional<ImpactAssessment>, review: Optional<PostIncidentReview>,
  signoffDecision: Optional<SignoffDecision>, status: PIRStatus, createdAt: Instant,
  completedAt: Optional<Instant>}. PIRStatus enum: CREATED, GATHERING, GATHERED, ASSESSING,
  ASSESSED, DRAFTING, DRAFTED, AWAITING_SIGNOFF, COMPLETE, REJECTED, FAILED. Events:
  PIRCreated{incidentId}, GatherStarted, EvidenceGathered{evidenceLog}, AssessStarted,
  ImpactAssessed{impactAssessment}, DraftStarted, ReviewDrafted{review},
  GuardrailRejected{check, reason}, SignoffRequested{assigneeEmail}, SignoffRecorded{decision},
  ReviewComplete, ReviewRejected{rejectionComment}, PIRFailed{reason}.
  Commands: create, startGather, recordEvidence, startAssess, recordAssessment, startDraft,
  recordReview, recordGuardrailRejection, requestSignoff, recordSignoff, complete, reject,
  fail, getPIR. emptyState() returns PIRRecord.initial("") with all Optional fields as
  Optional.empty() and no commandContext() reference (Lesson 3).

- 1 View PIRView with row type PIRRow that mirrors PIRRecord exactly (all Optional<T>
  lifecycle fields preserved). Table updater consumes PIREntity events. ONE query
  getAllPIRs: SELECT * AS reviews FROM pir_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * PIREndpoint at /api with POST /pir (body {incidentId}; mints pirId; calls
    PIREntity.create(incidentId); then starts PIRWorkflow with id "pir-" + pirId; returns
    {pirId}), GET /pir (list from getAllPIRs, sorted newest-first), GET /pir/{id} (one row),
    POST /pir/{id}/signoff (body {approved, comments}; calls PIREntity.recordSignoff then
    PIRWorkflow.resumeSignoff), GET /pir/sse (Server-Sent Events forwarded from the view's
    stream-updates), and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- PIRTasks.java declaring three Task<R> constants:
    GATHER_EVIDENCE = Task.name("Gather evidence").description("Fetch the incident record
      and all timeline events for this incident").resultConformsTo(EvidenceLog.class);
    ASSESS_IMPACT = Task.name("Assess impact").description("Classify impact severity and
      identify the root cause from the evidence log").resultConformsTo(ImpactAssessment.class);
    DRAFT_REVIEW = Task.name("Draft review").description("Compose a PostIncidentReview with
      executive summary, timeline, root cause, and action items").resultConformsTo(
      PostIncidentReview.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- GatherTools.java — @FunctionTool fetchIncidentRecord(String incidentId) -> IncidentRecord
  reading from src/main/resources/sample-data/incidents/<incidentId>.json; @FunctionTool
  fetchTimelineEvents(String incidentId) -> List<TimelineEvent> reading from the same file's
  "timeline" array.

- AssessTools.java — @FunctionTool classifyImpact(EvidenceLog evidenceLog) ->
  ImpactClassification (deterministic — derive severity from incident.severity, affectedSystems
  from timeline actor set, usersAffected and outageWindow from incident record);
  @FunctionTool identifyRootCause(EvidenceLog evidenceLog) -> RootCause (deterministic
  clustering — longest-repeated actor sequence in timeline becomes root cause summary;
  contributing factors are the distinct descriptions).

- DraftTools.java — @FunctionTool composeExecutiveSummary(ImpactAssessment assessment) ->
  String (1–3 sentence summary of severity, systems, and outage window); @FunctionTool
  buildActionItems(ImpactAssessment assessment, RootCause rootCause) -> List<ActionItem>
  (one ActionItem per contributing factor, plus one for the root cause itself; owners are
  sampled from a static on-call rotation list in the sample-data; due dates are 7/14/30 days
  from gatheredAt depending on priority).

- PIRGuardrail.java — implements the before-agent-response hook. Checks: (1) executive summary
  non-empty and >= 50 chars; (2) actionItems.size() >= 1 with non-null owner and dueDate on
  each; (3) every TimelineEvent.eventId referenced in review body is present in EvidenceLog
  (looked up via pirId in metadata); (4) review.impactClassification.severity matches
  evidenceLog.incident.severity. On reject returns Guardrail.reject("draft-quality-violation:
  <check> failed: <detail>") AND calls PIREntity.recordGuardrailRejection(check, reason) so
  the violation is visible in the UI's rejection-log strip.

- PIRSignoffNotifier.java — pure side-effect class. notify(pirId, assigneeEmail) writes a
  JSON stub notification to src/main/resources/sample-data/notifications/<pirId>.json (creates
  the file). In a production deployment, replace the stub with a real webhook or email sender.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9423 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/incidents.jsonl with 5 seeded incident lines covering the
  three surfaces named in J1-J4 plus two extras.

- src/main/resources/sample-data/incidents/*.json — three files keyed by seeded incident ID,
  each carrying an IncidentRecord + 4-8 TimelineEvent entries with deterministic content so
  GatherTools.fetchIncidentRecord and fetchTimelineEvents return the same data across restarts.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, H1) matching the mechanisms in
  Section 8 of this SPEC.

- risk-survey.yaml at the project root pre-filled for ops-automation domain (sector:
  technology-operations, decisions: authority_level: recommend-only, data_classes: pii: true
  because incident records may contain employee names, oversight: human_in_loop: true,
  agent_count: 1, agent_pattern: sequential-pipeline).

- prompts/pir-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Post-Incident Review Agent",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of PIR cards; right = selected-review detail with incident record, timeline,
  impact assessment, full review document, sign-off panel, guardrail-rejection strip).
  Browser title exactly: <title>Akka Sample: Post-Incident Review Agent</title>. No subtitle
  on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        outputs per Task. Sets model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java with per-task dispatch on Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json, picks one entry pseudo-randomly per call
  (seedFor(pirId)), deserialises into the task's typed return, and drives tool-call sequences.
- Per-task mock-response shapes:
    gather-evidence.json — 5 EvidenceLog entries, each with an IncidentRecord + 4-8
      TimelineEvent items. tool_calls: fetchIncidentRecord + fetchTimelineEvents in order.
    assess-impact.json — 5 ImpactAssessment entries paired with gather entries, each with
      ImpactClassification + RootCause. tool_calls: classifyImpact + identifyRootCause.
    draft-review.json — 5 PostIncidentReview entries. Each carries a non-empty
      executiveSummary, ≥ 2 ActionItems with owner+dueDate, timeline entries from the paired
      EvidenceLog, and tool_calls: composeExecutiveSummary + buildActionItems.
      Plus 1 deliberately INCOMPLETE entry missing actionItems (empty list) — PIRGuardrail
      rejects it; J2 verifies this. The mock selects the incomplete entry on the first
      iteration of every 4th review (modulo seed).
- A MockModelProvider.seedFor(pirId) helper makes per-review selection deterministic.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. PIRAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion PIRTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (gatherStep
  60s, assessStep 60s, draftStep 90s, signoffStep no timeout — HITL waits indefinitely,
  error step 5s).
- Lesson 6: every nullable lifecycle field on PIRRecord is Optional<T>.
- Lesson 7: PIRTasks.java with GATHER_EVIDENCE, ASSESS_IMPACT, DRAFT_REVIEW constants is
  mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build".
- Lesson 10: port 9423 declared explicitly in application.conf.
- Lesson 11: no source-platform metadata anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words.
- Lesson 24: static-resources/index.html includes mermaid CSS overrides AND themeVariables.
- Lesson 25: never write API key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: exactly ONE AutonomousAgent (PIRAgent).
- The sequential-pipeline invariant: GATHER → ASSESS → DRAFT task dependency carried by typed
  task results written to PIREntity between steps.
- The HITL sign-off is an application-layer pause, not an LLM call. PIRSignoffNotifier
  produces a side-effect; the workflow waits for an external HTTP POST to resume.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
