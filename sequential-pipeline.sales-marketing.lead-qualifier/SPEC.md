# SPEC — lead-qualifier

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Lead Qualifier.
**One-line pitch:** A user submits a sales inquiry; one `InquiryAgent` walks it through three task phases — **CAPTURE** contact and intent data, **QUALIFY** the lead with a scored profile, **ENRICH** the CRM — with each phase gated on the prior phase's recorded output, PII masked before any boundary-crossing tool call, and every CRM write validated for schema and scope before execution.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a sales-and-marketing domain. One `InquiryAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the CAPTURE task's typed output becomes the QUALIFY task's instruction context; the QUALIFY task's typed output becomes the ENRICH task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Two governance mechanisms are wired around the pipeline:

- A **`before-tool-call` guardrail** sits between the agent and every tool call. It looks at the call's declared phase (`CAPTURE` / `QUALIFY` / `ENRICH`) and the current `LeadEntity` status. An ENRICH-phase tool called while the entity has not yet recorded `QualificationScored` is rejected before the tool body runs. The rejection returns a structured error to the agent so the task loop can correct course inside its iteration budget. Beyond the phase-gate role, the same hook validates every CRM write for schema conformance (required fields present, stage value in the allowed enum) and scope (lead ID matches the in-flight workflow — no cross-lead writes). The guardrail also delegates PII masking to `PiiSanitizer` before any tool call that carries personal contact fields.
- A **`pii` sanitizer** (`PiiSanitizer`) intercepts outbound payloads on every tool call that crosses the service boundary. In this blueprint, the ENRICH-phase enrich tools carry the `LeadScore` and contact fields; `PiiSanitizer` replaces email with a hash, phone with a placeholder, and full name with initials before those fields are passed to the CRM tools. The original PII lives only on `LeadEntity` and never leaves the service in plaintext.

The blueprint shows that a sequential pipeline is not just a chain of LLM calls — the task-boundary handoffs are the right cut to enforce the dependency contract, and the `before-tool-call` hook is the right cut to enforce both the phase order and the outbound data-handling policy.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **raw inquiry** into the input (or picks one of three seeded inquiries — `Enterprise SaaS pricing for 500 seats`, `Integration partner referral – ERP migration project`, `Demo request: compliance reporting module`).
2. The user clicks **Qualify lead**. The UI POSTs to `/api/leads` and receives a `leadId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `CAPTURING` — the workflow has started `captureStep` and the agent has been handed the CAPTURE task.
4. Within ~10–20 s the card reaches `QUALIFYING` — the typed `InquiryForm` is visible in the card detail (a small table of extracted fields: name-initial, company, intent, product interest, channel). The agent's CAPTURE task returned; the workflow recorded `InquiryCaptured` and ran the QUALIFY task.
5. Within ~10–20 s more the card reaches `ENRICHING`. The `LeadScore` is visible (fit score, urgency tier, recommended stage, disqualification reason if any).
6. Within ~10–20 s more the card reaches `EVALUATED`. The right pane now shows the full typed `CrmEntry` — stage, owner candidate, notes, and a PII-masked contact block — plus a data-quality score chip (1–5) and a one-line rationale.
7. The user can submit another inquiry; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `LeadEndpoint` | `HttpEndpoint` | `/api/leads/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `LeadEntity`, `LeadView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `LeadEntity` | `EventSourcedEntity` | Per-lead lifecycle: created → capturing → captured → qualifying → qualified → enriching → enriched → evaluated. Source of truth. | `LeadEndpoint`, `LeadPipelineWorkflow` | `LeadView` |
| `LeadPipelineWorkflow` | `Workflow` | One workflow per lead. Steps: `captureStep` → `qualifyStep` → `enrichStep` → `evalStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `LeadEndpoint` after `CREATED` | `InquiryAgent`, `LeadEntity` |
| `InquiryAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `LeadTasks.java`: `CAPTURE_INQUIRY` → `InquiryForm`, `QUALIFY_LEAD` → `LeadScore`, `ENRICH_CRM` → `CrmEntry`. Each task is registered with the phase-appropriate function tools. | invoked by `LeadPipelineWorkflow` | returns typed results |
| `CaptureTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `parseContact(rawText)` and `detectIntent(rawText)`. Reads from `src/main/resources/sample-data/inquiries/*.json` for deterministic offline output. | called from CAPTURE task | returns `ContactFields` / `IntentSignals` |
| `QualifyTools` | function-tools class | Implements `scoreFit(form)` and `classifyUrgency(form)`. Pure in-memory scoring heuristics. | called from QUALIFY task | returns `FitScore` / `UrgencyTier` |
| `EnrichTools` | function-tools class | Implements `writeCrmStage(entry)` and `assignOwner(score)`. Simulated CRM writes reading from `sample-data/owners.json`. | called from ENRICH task | returns `CrmStageResult` / `OwnerAssignment` |
| `CrmWriteGuardrail` | `before-tool-call` guardrail (registered on `InquiryAgent`) | Reads the in-flight task's declared phase and the current `LeadEntity.status`. Rejects any tool call whose phase precondition has not been satisfied. For ENRICH-phase tools, additionally validates schema (required CRM fields present, stage value in allowed enum) and scope (leadId match). Delegates PII masking to `PiiSanitizer` before the tool call proceeds. | every tool call on every task | accept / structured-reject |
| `PiiSanitizer` | plain class (no Akka primitive) | Intercepts outbound payloads on ENRICH-phase tool calls. Replaces email with SHA-256 prefix, phone with `[REDACTED]`, and full name with initials. | called from `CrmWriteGuardrail` on ENRICH tool calls | masked payload or original payload |
| `QualityScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `CrmEntry`, `LeadScore`, `InquiryForm`. Output: `EvalResult{score, rationale}`. | called from `evalStep` | returns score |
| `LeadView` | `View` | Read model: one row per lead for the UI. | `LeadEntity` events | `LeadEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ContactFields(
    String nameInitial,       // PII-masked version used downstream
    String rawEmail,          // stored only on LeadEntity, never leaves service
    String rawPhone,          // stored only on LeadEntity, never leaves service
    String company,
    String channel            // web-form | email | referral | event
) {}

record IntentSignals(
    String productInterest,
    String useCaseSummary,
    String budgetIndicator    // unknown | <10k | 10k-50k | 50k+
) {}

record InquiryForm(
    ContactFields contact,
    IntentSignals intent,
    String rawText,
    Instant capturedAt
) {}

record FitScore(
    int score,                // 0-100
    String reasoning
) {}

enum UrgencyTier { HIGH, MEDIUM, LOW, DISQUALIFIED }

record LeadScore(
    FitScore fit,
    UrgencyTier urgency,
    String recommendedStage,  // New | Qualified | MQL | SQL | Disqualified
    Optional<String> disqualificationReason,
    Instant scoredAt
) {}

record CrmStageResult(
    String leadId,
    String stage,
    boolean written
) {}

record OwnerAssignment(
    String ownerId,
    String ownerName,
    String rationale
) {}

record CrmEntry(
    String leadId,
    String stage,
    String ownerCandidate,
    String notes,
    String maskedEmail,       // SHA-256 prefix of rawEmail
    String maskedPhone,       // "[REDACTED]"
    String nameInitial,
    Instant enrichedAt
) {}

record EvalResult(
    int score,                // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record LeadRecord(
    String leadId,
    Optional<String> rawInquiry,
    Optional<InquiryForm> form,
    Optional<LeadScore> score,
    Optional<CrmEntry> crmEntry,
    Optional<EvalResult> eval,
    LeadStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum LeadStatus {
    CREATED, CAPTURING, CAPTURED, QUALIFYING, QUALIFIED,
    ENRICHING, ENRICHED, EVALUATED, FAILED
}
```

Events on `LeadEntity`: `LeadCreated`, `CaptureStarted`, `InquiryCaptured`, `QualifyStarted`, `QualificationScored`, `EnrichStarted`, `CrmEntryWritten`, `EvaluationScored`, `GuardrailRejected`, `LeadFailed`.

Every nullable lifecycle field on the `LeadRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/leads` — body `{ rawInquiry }` → `{ leadId }`.
- `GET /api/leads` — list all leads, newest-first.
- `GET /api/leads/{id}` — one lead.
- `GET /api/leads/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Lead Qualifier</title>`.

The App UI tab is a two-column layout: a left rail with the live list of leads (status pill + company + age) and a right pane with the selected lead's detail — raw inquiry text, captured form fields, lead score panel, CRM entry block, data-quality score chip, and a guardrail-rejection log strip if any phase-gate or schema rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-tool-call` guardrail (CRM write gate)**: `CrmWriteGuardrail` is registered on `InquiryAgent` and runs before every tool call. It reads the in-flight `Task`'s declared phase (encoded as a constant on each function-tool class — `Phase.CAPTURE`, `Phase.QUALIFY`, `Phase.ENRICH`) and the current `LeadEntity.status` for the lead the task is bound to. The accept rule is precise: `CAPTURE` tools require `status ∈ {CREATED, CAPTURING}`; `QUALIFY` tools require `status ∈ {CAPTURED, QUALIFYING}` AND `form.isPresent()`; `ENRICH` tools require `status ∈ {QUALIFIED, ENRICHING}` AND `score.isPresent()`. For ENRICH-phase tools that write to CRM (`writeCrmStage`, `assignOwner`), the guardrail additionally validates that all required CRM fields are present (`stage`, `leadId`, `ownerCandidate`) and that `stage` is in the allowed enum `{New, Qualified, MQL, SQL, Disqualified}`. A stage value outside that enum is rejected with a `schema-violation` error. On reject, the guardrail returns a structured error to the agent loop and the workflow records a `GuardrailRejected{phase, tool, reason}` event for visibility. The agent loop retries within its 4-iteration budget. Before any accepted ENRICH-phase tool call proceeds, the guardrail delegates PII masking to `PiiSanitizer`.
- **S1 — `pii` sanitizer**: `PiiSanitizer` is called from `CrmWriteGuardrail` immediately before any ENRICH-phase tool call proceeds. It receives the outbound payload, replaces `rawEmail` with the first 8 hex chars of its SHA-256 digest followed by `@masked`, replaces `rawPhone` with `[REDACTED]`, and replaces `fullName` with initials only. The masked payload is what the tool body receives; the original PII remains exclusively on `LeadEntity` and is never included in any log line, tool argument, or SSE event emitted to the UI.

## 9. Agent prompts

- `InquiryAgent` → `prompts/inquiry-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output. In the ENRICH phase, the agent is instructed not to include raw PII in any field it constructs — all personal identifiers must come from the masked `CrmEntry` template provided in the task's instruction context.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded inquiry `Enterprise SaaS pricing for 500 seats`; within 60 s the lead reaches `EVALUATED` with a non-null `InquiryForm`, a `LeadScore` with `urgency ∈ {HIGH, MEDIUM}`, a `CrmEntry` with a valid stage and owner, and a data-quality score chip on the card.
2. **J2** — The agent's first iteration on a lead calls an ENRICH-phase tool (`writeCrmStage`) before `QualificationScored` has been recorded (mock LLM path). `CrmWriteGuardrail` rejects the call; a `GuardrailRejected` event lands on the entity; the agent retries in-phase; the lead eventually completes correctly. The UI's rejection-log strip shows the one rejected call.
3. **J3** — A lead whose mock-LLM trajectory constructs a `CrmEntry` with an invalid stage value (`stage = "Prospect"`) is rejected by the schema check; a second `GuardrailRejected` event lands; the agent corrects to `stage = "New"` and the pipeline completes. The rejection-log strip shows two rejections.
4. **J4** — The `CrmEntry` surfaced in the UI carries no raw email, no raw phone, and no full name — only `maskedEmail`, `[REDACTED]` phone, and name initials. The raw fields are absent from every SSE event payload, every log line at INFO or below, and every API response body.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named lead-qualifier demonstrating the sequential-pipeline x sales-marketing
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-sales-marketing-lead-qualifier.
Java package io.akka.samples.aidrivenleadmanagementandinquiryautomationwitherpnext.
Akka 3.6.0. HTTP port 9612.

Components to wire (exactly):

- 1 AutonomousAgent InquiryAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/inquiry-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  CAPTURE, QUALIFY, and ENRICH tool sets are ALL registered on the agent; phase gating is the
  job of CrmWriteGuardrail, NOT of conditional .tools(...) wiring. The before-tool-call
  guardrail (CrmWriteGuardrail) is registered on the agent via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries within its
  4-iteration budget.

- 1 Workflow LeadPipelineWorkflow per leadId with four steps:
  * captureStep — emits CaptureStarted on the entity, then calls componentClient
    .forAutonomousAgent(InquiryAgent.class, "agent-" + leadId).runSingleTask(
      TaskDef.instructions("Raw inquiry: " + rawInquiry + "\nPhase: CAPTURE\nExtract contact
      fields and intent signals from the raw inquiry text.")
        .metadata("leadId", leadId)
        .metadata("phase", "CAPTURE")
        .taskType(LeadTasks.CAPTURE_INQUIRY)
    ). Reads forTask(taskId).result(CAPTURE_INQUIRY) to get InquiryForm. Writes
    LeadEntity.recordForm(form). WorkflowSettings.stepTimeout 60s.
  * qualifyStep — emits QualifyStarted, then runSingleTask with TaskDef.instructions
    (formatQualifyContext(form, rawInquiry)) and metadata.phase = "QUALIFY", taskType
    QUALIFY_LEAD. Writes LeadEntity.recordScore(score). stepTimeout 60s.
  * enrichStep — emits EnrichStarted, then runSingleTask with TaskDef.instructions
    (formatEnrichContext(score, form, rawInquiry)) and metadata.phase = "ENRICH", taskType
    ENRICH_CRM. Writes LeadEntity.recordCrmEntry(crmEntry). stepTimeout 60s.
  * evalStep — runs the deterministic QualityScorer over (crmEntry, score, form) and writes
    LeadEntity.recordEvaluation(eval). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(LeadPipelineWorkflow::error). The error step writes
  LeadFailed and ends.

- 1 EventSourcedEntity LeadEntity (one per leadId). State LeadRecord{leadId,
  rawInquiry: Optional<String>, form: Optional<InquiryForm>, score: Optional<LeadScore>,
  crmEntry: Optional<CrmEntry>, eval: Optional<EvalResult>, status: LeadStatus, createdAt:
  Instant, finishedAt: Optional<Instant>}. LeadStatus enum: CREATED, CAPTURING, CAPTURED,
  QUALIFYING, QUALIFIED, ENRICHING, ENRICHED, EVALUATED, FAILED. Events:
  LeadCreated{rawInquiry}, CaptureStarted, InquiryCaptured{form}, QualifyStarted,
  QualificationScored{score}, EnrichStarted, CrmEntryWritten{crmEntry},
  EvaluationScored{eval}, GuardrailRejected{phase, tool, reason}, LeadFailed{reason}.
  Commands: create, startCapture, recordForm, startQualify, recordScore, startEnrich,
  recordCrmEntry, recordEvaluation, recordGuardrailRejection, fail, getLead. emptyState()
  returns LeadRecord.initial("") with all Optional fields as Optional.empty() and no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 View LeadView with row type LeadRow that mirrors LeadRecord exactly (all Optional<T>
  lifecycle fields preserved). Table updater consumes LeadEntity events. ONE query
  getAllLeads: SELECT * AS leads FROM lead_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * LeadEndpoint at /api with POST /leads (body {rawInquiry}; mints leadId; calls
    LeadEntity.create(rawInquiry); then starts LeadPipelineWorkflow with id
    "pipeline-" + leadId; returns {leadId}), GET /leads (list from getAllLeads,
    sorted newest-first), GET /leads/{id} (one row), GET /leads/sse (Server-Sent
    Events forwarded from the view's stream-updates), and three /api/metadata/*
    endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- LeadTasks.java declaring three Task<R> constants:
    CAPTURE_INQUIRY = Task.name("Capture inquiry").description("Extract contact fields and
      intent signals from the raw inquiry text").resultConformsTo(InquiryForm.class);
    QUALIFY_LEAD = Task.name("Qualify lead").description("Score the captured inquiry for fit
      and urgency; assign a recommended CRM stage").resultConformsTo(LeadScore.class);
    ENRICH_CRM = Task.name("Enrich CRM").description("Write the qualified lead into CRM
      stages and assign an owner candidate").resultConformsTo(CrmEntry.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Phase.java — enum {CAPTURE, QUALIFY, ENRICH}. Each function-tool method is annotated with
  the constant phase (use a custom annotation if the SDK's @FunctionTool does not carry a
  phase field — the guardrail reads it from a parallel registry built at startup if so).

- CaptureTools.java — @FunctionTool parseContact(String rawText) -> ContactFields reading
  from src/main/resources/sample-data/inquiries/*.json keyed by inquiry id; @FunctionTool
  detectIntent(String rawText) -> IntentSignals reading from the matching inquiry entry.

- QualifyTools.java — @FunctionTool scoreFit(InquiryForm form) -> FitScore (deterministic
  scoring: company non-empty +20, budgetIndicator "50k+" +40, productInterest non-empty +20,
  channel "referral" +20); @FunctionTool classifyUrgency(InquiryForm form) -> UrgencyTier
  (HIGH if fit.score >= 70, MEDIUM if >= 40, DISQUALIFIED if intent keywords match
  disqualify list, else LOW).

- EnrichTools.java — @FunctionTool writeCrmStage(CrmEntry entry) -> CrmStageResult
  (validates stage in {New, Qualified, MQL, SQL, Disqualified}, writes to
  sample-data/crm-log.json); @FunctionTool assignOwner(LeadScore score) -> OwnerAssignment
  (reads from sample-data/owners.json, picks owner by urgency tier round-robin).

- CrmWriteGuardrail.java — implements the before-tool-call hook. Reads the candidate tool
  call's phase attribute, looks up the LeadEntity status by leadId (carried in the TaskDef
  metadata), applies the accept matrix from Section 8. For ENRICH-phase CRM-write tools,
  additionally validates schema (required fields present, stage in allowed enum). Delegates
  to PiiSanitizer before passing the masked payload to the tool. On reject ALSO calls
  LeadEntity.recordGuardrailRejection(phase, tool, reason).

- PiiSanitizer.java — pure deterministic logic (no LLM). Inputs: outbound tool payload.
  Outputs: masked payload. Replaces rawEmail with sha256(rawEmail).substring(0,8) + "@masked",
  rawPhone with "[REDACTED]", fullName fields with initials. Never logs raw PII. Called
  exclusively from CrmWriteGuardrail.

- QualityScorer.java — pure deterministic logic (no LLM). Inputs: CrmEntry, LeadScore,
  InquiryForm. Outputs: EvalResult with score and rationale. Four checks, one point per
  check satisfied, starting from a base of 1: stage validity (stage in allowed enum),
  owner assignment (ownerCandidate non-empty), notes present (notes.length() > 0), and
  fit-score consistency (crmEntry.stage matches expected stage for score.urgency). Score
  range 1-5. Rationale names the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9612 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/inquiries.jsonl with 5 seeded inquiry lines covering the
  three surfaces named in J1-J5 plus two extras.

- src/main/resources/sample-data/inquiries/*.json — three files keyed by seeded inquiry id,
  each carrying deterministic ContactFields + IntentSignals so CaptureTools returns the
  same output across restarts.

- src/main/resources/sample-data/owners.json — list of 4-6 owner entries each with ownerId,
  ownerName, urgencyAffinities list.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, S1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors — sample domain.

- risk-survey.yaml at the project root with data.data_classes.pii = true (lead contact info
  is personal data), decisions.authority_level = recommend-only (a human SDR reviews the
  qualified lead before CRM is treated as authoritative), oversight.human_in_loop = true,
  operations.agent_count = 1, operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "crm-write-with-invalid-stage", "pii-leak-in-tool-payload",
  "premature-crm-write", "disqualified-lead-routed"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/inquiry-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Lead Qualifier", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of lead cards; right = selected-lead detail with raw inquiry header, form fields
  table, lead score panel, CRM entry block, data-quality score chip, rejection-log strip).
  Browser title exactly: <title>Akka Sample: Lead Qualifier</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task (see Mock LLM provider block below).
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java with per-task dispatch on the Task<R> id. Each branch
  reads src/main/resources/mock-responses/<task-id>.json, picks one entry pseudo-randomly
  per call (seedFor(leadId)), and deserialises into the task's typed return.
- Per-task mock-response shapes:
    capture-inquiry.json — 5 InquiryForm entries, each with realistic ContactFields +
      IntentSignals keyed to the three seeded inquiries. Each entry's tool_calls array
      contains 2 calls: parseContact(rawText) + detectIntent(rawText). Plus 1
      deliberately PHASE-VIOLATING entry whose tool_calls array starts with
      writeCrmStage(...) (an ENRICH-phase tool called during the CAPTURE phase) —
      the guardrail rejects it. The mock selects the violating entry on the FIRST
      iteration of every 3rd lead (modulo seed) so J2 is reproducible.
    qualify-lead.json — 5 LeadScore entries paired one-to-one with the capture entries,
      each with FitScore and UrgencyTier, tool_calls containing scoreFit +
      classifyUrgency in order.
    enrich-crm.json — 5 CrmEntry entries paired one-to-one. Each carries valid stage,
      ownerCandidate, notes, maskedEmail, maskedPhone. Plus 1 entry with stage =
      "Prospect" (invalid) — guardrail rejects the schema violation (J3).
- A MockModelProvider.seedFor(leadId) helper makes per-lead selection deterministic.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. InquiryAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion LeadTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (captureStep
  60s, qualifyStep 60s, enrichStep 60s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on LeadRecord is Optional<T>.
- Lesson 7: LeadTasks.java with CAPTURE_INQUIRY, QUALIFY_LEAD, ENRICH_CRM is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9612 declared explicitly in application.conf.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words.
- Lesson 24: mermaid CSS overrides and themeVariables block in index.html.
- Lesson 25: NEVER write the key value to disk.
- Lesson 26: tab switching by data-tab / data-panel attribute, never by NodeList index.
- The single-agent invariant: exactly ONE AutonomousAgent (InquiryAgent). QualityScorer and
  PiiSanitizer are rule-based and do NOT make LLM calls.
- The sequential-pipeline invariant: all tool sets registered on the agent; CrmWriteGuardrail
  is the runtime gate. Do NOT conditionally register tools per task.
- PII invariant: rawEmail and rawPhone NEVER appear in any outbound tool payload, log line
  at INFO or below, SSE event, or API response body. PiiSanitizer runs before every
  ENRICH-phase tool call.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous.
2. Run `/akka:tasks` — break the plan into implementation tasks.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
