# SPEC — cold-outreach

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** LeadPilot Lite - Personalized Cold Email.
**One-line pitch:** A user submits a prospect record; one `OutreachAgent` walks it through three task phases — **RESEARCH** firmographic and intent signals, **DRAFT** a personalized email body, **SEND** after a mandatory human-approval gate — with each phase gated on the prior phase's recorded output and the send tool blocked until an explicit reviewer decision is present.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a sales-marketing domain. One `OutreachAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the RESEARCH task's typed output becomes the DRAFT task's instruction context; the DRAFT task's typed output becomes the SEND task's instruction context only after the HITL approval gate clears. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Three governance mechanisms are wired around the pipeline:

- A **`before-tool-call` guardrail** (`SendGuardrail`) sits between the agent and `sendEmail`. It reads the current `ProspectEntity` status and verifies that a `ReviewDecided{approved: true}` event has been recorded for this prospect. If the entity does not have an approved review decision, the guardrail rejects the call before the tool body runs, returns a structured `send-blocked` error to the agent loop, and records a `GuardrailRejected` event on the entity. This prevents any send that bypasses the approval workflow — including mock-LLM paths that call `sendEmail` during the DRAFT phase.

- A **`before-agent-response` guardrail** (`ComplianceGuardrail`) runs on every email body the agent proposes to return at the end of the DRAFT task. It checks three deterministic CAN-SPAM/GDPR rules: (1) the body contains an unsubscribe notice (`[unsubscribe]` sentinel or equivalent), (2) the prospect's country code is not on the block list for the deployer's licence (checked against `src/main/resources/compliance/blocked-regions.json`), and (3) the sender's physical address is present in the body. Any failure causes the guardrail to return a structured `compliance-violation` error listing each failed rule; the agent loop revises the draft within its 4-iteration budget.

- A **human-in-the-loop (HITL) approval gate** runs as `approvalStep` inside the workflow, after `EmailDrafted` lands and before `sendStep` starts. The workflow pauses and emits a `ReviewRequested` event on the entity. A reviewer calls `POST /api/outreach/{id}/review` with `{ approved: boolean, reason: string }`. The workflow resumes when the entity records `ReviewDecided`. If rejected, the pipeline ends with `REVIEW_REJECTED`; no send occurs.

The blueprint shows that a sequential pipeline can enforce both phase isolation and an external human decision without collapsing into a single unbounded agent loop. The task-boundary handoffs are the right cut for both the dependency contract and the per-phase tool isolation.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **prospect** (company name and contact email) into the input (or picks one of three seeded prospects — `Acme DevTools <dev@acme.example>`, `BrightPath Analytics <ops@brightpath.example>`, `Meridian Cloud <founders@meridian.example>`).
2. The user clicks **Run outreach**. The UI POSTs to `/api/outreach` and receives a `prospectId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `RESEARCHING` — the workflow has started `researchStep` and the agent has been handed the RESEARCH task.
4. Within ~10–20 s the card reaches `RESEARCHED` — the typed `ProspectProfile` is visible in the card detail (company summary, signals table with firmographic and intent items). The agent's RESEARCH task returned; the workflow recorded `ProspectResearched` and ran the DRAFT task.
5. Within ~10–20 s more the card reaches `AWAITING_REVIEW`. The `EmailDraft` is visible (subject line, body, personalisation fields). The pipeline is paused; a yellow "Review required" banner appears. The reviewer can click **Approve** or **Reject**.
6. After the reviewer clicks **Approve**, the card transitions to `SENDING`. Within ~5 s it reaches `SENT`. The right pane shows the full typed `OutreachEmail` — recipient, subject, body, personalisation metadata, sent timestamp, and compliance-check result.
7. If the reviewer clicks **Reject**, the card transitions to `REVIEW_REJECTED`. No email is sent. The rejection reason is shown.
8. The user can submit another prospect; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `OutreachEndpoint` | `HttpEndpoint` | `/api/outreach/*` — submit, list, get, review, SSE; serves `/api/metadata/*`. | — | `ProspectEntity`, `ProspectView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ProspectEntity` | `EventSourcedEntity` | Per-prospect outreach lifecycle: created → researching → researched → drafting → drafted → awaiting-review → review-decided → sending → sent / review-rejected. Source of truth. | `OutreachEndpoint`, `OutreachPipelineWorkflow` | `ProspectView` |
| `OutreachPipelineWorkflow` | `Workflow` | One workflow per prospect. Steps: `researchStep` → `draftStep` → `approvalStep` → `sendStep`. Each LLM-calling step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. `approvalStep` waits for a human review decision before proceeding. | started by `OutreachEndpoint` after `CREATED` | `OutreachAgent`, `ProspectEntity` |
| `OutreachAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `OutreachTasks.java`: `RESEARCH_PROSPECT` → `ProspectProfile`, `DRAFT_EMAIL` → `EmailDraft`, `SEND_EMAIL` → `OutreachEmail`. Each task is registered with the phase-appropriate function tools. | invoked by `OutreachPipelineWorkflow` | returns typed results |
| `ResearchTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `lookupFirmographics(company)` and `fetchIntentSignals(domain)`. Reads from `src/main/resources/sample-data/prospects/*.json` for deterministic offline output. | called from RESEARCH task | returns `FirmographicData` / `List<IntentSignal>` |
| `DraftTools` | function-tools class | Implements `personalizeLine(profile, field)` and `renderTemplate(template, vars)`. Pure in-memory transformations against `src/main/resources/templates/email-templates.json`. | called from DRAFT task | returns `String` (personalized line) / `EmailDraft` |
| `SendTools` | function-tools class | Implements `sendEmail(recipient, subject, body)`. In the sample, writes to `src/main/resources/sent-log/sent.jsonl` (no live SMTP). | called from SEND task | returns `SendReceipt` |
| `SendGuardrail` | `before-tool-call` guardrail (registered on `OutreachAgent`) | Reads the current `ProspectEntity` status and the `reviewDecision` field. Blocks `sendEmail` unless status is `SENDING` and `reviewDecision.approved == true`. On reject, records `GuardrailRejected` on the entity. | every tool call on every task | accept / structured-reject |
| `ComplianceGuardrail` | `before-agent-response` guardrail (registered on `OutreachAgent`) | Runs deterministic CAN-SPAM/GDPR checks on every email body the agent proposes to return during the DRAFT task. Checks: unsubscribe notice present, prospect region not blocked, physical sender address present. | every agent response during DRAFT task | accept / compliance-violation |
| `ProspectView` | `View` | Read model: one row per prospect for the UI. | `ProspectEntity` events | `OutreachEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record IntentSignal(String signalType, String description, Instant observedAt) {}

record FirmographicData(
    String companyName,
    String domain,
    String industry,
    int employeeCount,
    String hqCountry,
    List<IntentSignal> intentSignals,
    Instant researchedAt
) {}

record ProspectProfile(
    String prospectId,
    String contactEmail,
    FirmographicData firmographics,
    List<String> personalizationHooks,
    Instant profiledAt
) {}

record EmailDraft(
    String subject,
    String body,
    List<String> personalizationFields,
    String templateId,
    Instant draftedAt
) {}

record ComplianceCheckResult(
    boolean passed,
    List<String> failedRules,
    Instant checkedAt
) {}

record ReviewDecision(
    boolean approved,
    String reviewerNote,
    Instant decidedAt
) {}

record SendReceipt(
    String messageId,
    String recipient,
    Instant sentAt
) {}

record OutreachEmail(
    String subject,
    String body,
    String recipient,
    ComplianceCheckResult compliance,
    SendReceipt receipt,
    Instant finishedAt
) {}

record ProspectRecord(
    String prospectId,
    Optional<String> contactEmail,
    Optional<ProspectProfile> profile,
    Optional<EmailDraft> draft,
    Optional<ComplianceCheckResult> compliance,
    Optional<ReviewDecision> reviewDecision,
    Optional<OutreachEmail> outreachEmail,
    OutreachStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum OutreachStatus {
    CREATED, RESEARCHING, RESEARCHED, DRAFTING, DRAFTED,
    AWAITING_REVIEW, SENDING, SENT, REVIEW_REJECTED, FAILED
}
```

Events on `ProspectEntity`: `ProspectCreated`, `ResearchStarted`, `ProspectResearched`, `DraftStarted`, `EmailDrafted`, `ComplianceChecked`, `ReviewRequested`, `ReviewDecided`, `SendStarted`, `EmailSent`, `GuardrailRejected`, `OutreachFailed`.

Every nullable lifecycle field on the `ProspectRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/outreach` — body `{ contactEmail, companyName }` → `{ prospectId }`.
- `GET /api/outreach` — list all prospects, newest-first.
- `GET /api/outreach/{id}` — one prospect.
- `POST /api/outreach/{id}/review` — body `{ approved: boolean, reason: string }` → `{ prospectId, status }`.
- `GET /api/outreach/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: LeadPilot Lite - Personalized Cold Email</title>`.

The App UI tab is a two-column layout: a left rail with the live list of prospects (status pill + contact + company + age) and a right pane with the selected prospect's detail — research summary, intent signals, draft email with subject and body, compliance badge, HITL approval panel (Approve / Reject buttons, active only when status is `AWAITING_REVIEW`), and a guardrail-rejection log strip if any rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-tool-call` guardrail (send-gate)**: `SendGuardrail` is registered on `OutreachAgent` and runs before every tool call. It checks the candidate tool name — only `sendEmail` is gated. The accept rule: `sendEmail` requires `status ∈ {SENDING}` AND `reviewDecision.isPresent()` AND `reviewDecision.approved == true`. On reject, the guardrail returns a structured `send-blocked` error to the agent loop and calls `ProspectEntity.recordGuardrailRejection(phase, tool, reason)` so the rejection is visible in the UI's rejection-log strip and in the audit log. The agent loop retries within its 4-iteration budget.

- **H2 — `before-agent-response` guardrail (compliance check)**: `ComplianceGuardrail` is registered on `OutreachAgent` and runs on every proposed agent response during the DRAFT task. Three deterministic checks: (1) the email body contains an unsubscribe notice token (`[unsubscribe]`), (2) the prospect's `hqCountry` from `FirmographicData` is not in `blocked-regions.json`, and (3) the body contains the deployer's physical address sentinel (`[sender-address]`). Any failure returns a `compliance-violation` error listing each failed check by rule id. The agent loop revises the draft within its 4-iteration budget.

- **HITL1 — application-layer human approval**: `approvalStep` in `OutreachPipelineWorkflow` emits `ReviewRequested` on the entity and then pauses. The workflow waits for a `POST /api/outreach/{id}/review` call from a human reviewer. The endpoint calls `ProspectEntity.recordReviewDecision(decision)`, which emits `ReviewDecided`. The workflow resumes: if `approved == true` it advances to `sendStep`; if `approved == false` it transitions to `REVIEW_REJECTED` and ends without sending.

## 9. Agent prompts

- `OutreachAgent` → `prompts/outreach-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output. It also specifies that every `EmailDraft.body` must include the `[unsubscribe]` and `[sender-address]` sentinel tokens or the `before-agent-response` guardrail will reject the response.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded prospect `Acme DevTools <dev@acme.example>`; within 60 s the outreach reaches `SENT` with a non-empty `ProspectProfile`, a compliant `EmailDraft`, an approved `ReviewDecision`, and a `SendReceipt` on the card.
2. **J2** — The agent's first iteration on a prospect calls `sendEmail` before `ReviewDecided{approved: true}` has been recorded (mock LLM path). `SendGuardrail` rejects the call; a `GuardrailRejected` event lands on the entity; the pipeline pauses at `approvalStep` correctly; after manual approval the pipeline proceeds to `SENT`. The UI's rejection-log strip shows the one rejected call.
3. **J3** — The agent produces a draft body without the `[unsubscribe]` sentinel (mock LLM path). `ComplianceGuardrail` rejects the response; the agent revises the draft to include the sentinel; the revised draft passes. The card shows the compliance badge as `PASSED`.
4. **J4** — A reviewer submits `{ approved: false, reason: "wrong tone" }` → the entity transitions to `REVIEW_REJECTED` → no `EmailSent` event → no `sendEmail` tool call in the audit log. The card shows the rejection reason.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named cold-outreach demonstrating the sequential-pipeline x sales-marketing cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-sales-marketing-cold-outreach. Java package
io.akka.samples.leadpilotlitepersonalizedcoldemail. Akka 3.6.0. HTTP port 9969.

Components to wire (exactly):

- 1 AutonomousAgent OutreachAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/outreach-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  RESEARCH, DRAFT, and SEND tool sets are ALL registered on the agent; phase gating is the
  job of SendGuardrail, NOT of conditional .tools(...) wiring. Both guardrails (SendGuardrail
  and ComplianceGuardrail) are registered on the agent via the agent's guardrail-configuration
  block. On guardrail rejection the agent loop retries within its 4-iteration budget.

- 1 Workflow OutreachPipelineWorkflow per prospectId with four steps:
  * researchStep — emits ResearchStarted on the entity, then calls componentClient
    .forAutonomousAgent(OutreachAgent.class, "agent-" + prospectId).runSingleTask(
      TaskDef.instructions("Prospect: " + contactEmail + " at " + companyName +
      "\nPhase: RESEARCH\nUse lookupFirmographics and fetchIntentSignals to build a
      ProspectProfile.")
        .metadata("prospectId", prospectId)
        .metadata("phase", "RESEARCH")
        .taskType(OutreachTasks.RESEARCH_PROSPECT)
    ). Reads forTask(taskId).result(RESEARCH_PROSPECT) to get ProspectProfile. Writes
    ProspectEntity.recordProfile(profile). WorkflowSettings.stepTimeout 60s.
  * draftStep — emits DraftStarted, then runSingleTask with TaskDef.instructions
    (formatDraftContext(profile, companyName)) and metadata.phase = "DRAFT", taskType
    DRAFT_EMAIL. Writes ProspectEntity.recordDraft(draft) and
    ProspectEntity.recordComplianceCheck(result). stepTimeout 60s.
  * approvalStep — emits ReviewRequested on the entity and then pauses. The step completes
    only when the entity records ReviewDecided (triggered by POST /api/outreach/{id}/review).
    WorkflowSettings.stepTimeout 86400s (24 h — reviewer SLA). On rejection
    (approved == false), transitions to REVIEW_REJECTED and ends. stepTimeout 86400s.
  * sendStep — emits SendStarted, then runSingleTask with TaskDef.instructions
    (formatSendContext(draft, profile)) and metadata.phase = "SEND", taskType SEND_EMAIL.
    Writes ProspectEntity.recordEmail(outreachEmail). stepTimeout 30s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(OutreachPipelineWorkflow::error). The error step writes
  OutreachFailed and ends.

- 1 EventSourcedEntity ProspectEntity (one per prospectId). State ProspectRecord{prospectId,
  contactEmail: Optional<String>, profile: Optional<ProspectProfile>, draft: Optional<EmailDraft>,
  compliance: Optional<ComplianceCheckResult>, reviewDecision: Optional<ReviewDecision>,
  outreachEmail: Optional<OutreachEmail>, status: OutreachStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. OutreachStatus enum: CREATED, RESEARCHING, RESEARCHED,
  DRAFTING, DRAFTED, AWAITING_REVIEW, SENDING, SENT, REVIEW_REJECTED, FAILED. Events:
  ProspectCreated{contactEmail, companyName}, ResearchStarted, ProspectResearched{profile},
  DraftStarted, EmailDrafted{draft}, ComplianceChecked{result}, ReviewRequested,
  ReviewDecided{decision}, SendStarted, EmailSent{outreachEmail}, GuardrailRejected{phase, tool,
  reason}, OutreachFailed{reason}. Commands: create, startResearch, recordProfile, startDraft,
  recordDraft, recordComplianceCheck, requestReview, recordReviewDecision, startSend,
  recordEmail, recordGuardrailRejection, fail, getProspect. emptyState() returns
  ProspectRecord.initial("") with all Optional fields as Optional.empty() and no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 View ProspectView with row type ProspectRow that mirrors ProspectRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes ProspectEntity events. ONE
  query getAllProspects: SELECT * AS prospects FROM prospect_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * OutreachEndpoint at /api with POST /outreach (body {contactEmail, companyName}; mints
    prospectId; calls ProspectEntity.create(contactEmail, companyName); then starts
    OutreachPipelineWorkflow with id "pipeline-" + prospectId; returns {prospectId}), POST
    /outreach/{id}/review (body {approved, reason}; calls ProspectEntity.recordReviewDecision;
    returns {prospectId, status}), GET /outreach (list from getAllProspects, sorted
    newest-first), GET /outreach/{id} (one row), GET /outreach/sse (Server-Sent Events
    forwarded from the view's stream-updates), and three /api/metadata/* endpoints serving
    the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- OutreachTasks.java declaring three Task<R> constants:
    RESEARCH_PROSPECT = Task.name("Research prospect").description("Gather firmographic data
      and intent signals for a prospect by calling lookupFirmographics and
      fetchIntentSignals").resultConformsTo(ProspectProfile.class);
    DRAFT_EMAIL = Task.name("Draft email").description("Compose a personalized email body
      using personalizeLine and renderTemplate; body MUST include [unsubscribe] and
      [sender-address] sentinels").resultConformsTo(EmailDraft.class);
    SEND_EMAIL = Task.name("Send email").description("Send the approved draft via sendEmail
      and return a SendReceipt").resultConformsTo(OutreachEmail.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- OutreachPhase.java — enum {RESEARCH, DRAFT, SEND}. Each function-tool method is annotated
  with its phase constant.

- ResearchTools.java — @FunctionTool lookupFirmographics(String company) -> FirmographicData
  reading from src/main/resources/sample-data/prospects/*.json keyed by company slug;
  @FunctionTool fetchIntentSignals(String domain) -> List<IntentSignal> reading from the
  matching prospect entry's intentSignals list.

- DraftTools.java — @FunctionTool personalizeLine(ProspectProfile profile, String field) ->
  String (template variable substitution from email-templates.json); @FunctionTool
  renderTemplate(String templateId, Map<String,String> vars) -> EmailDraft (assembles subject
  + body from the named template with vars substituted; always appends [unsubscribe] and
  [sender-address] sentinels).

- SendTools.java — @FunctionTool sendEmail(String recipient, String subject, String body) ->
  SendReceipt (appends to src/main/resources/sent-log/sent.jsonl; mints messageId as
  "msg-" + sha1(recipient + subject).substring(0,8)).

- SendGuardrail.java — implements the before-tool-call hook. Only activates on the tool name
  `sendEmail`. Reads the ProspectEntity status and reviewDecision by prospectId (carried in
  the TaskDef metadata). Accept rule: status == SENDING AND reviewDecision.isPresent() AND
  reviewDecision.get().approved() == true. On reject, returns
  Guardrail.reject("send-blocked: sendEmail requires an approved review decision, saw status
  <status> and reviewDecision <present/absent>"). Also calls
  ProspectEntity.recordGuardrailRejection(phase, tool, reason).

- ComplianceGuardrail.java — implements the before-agent-response hook. Only activates during
  the DRAFT task (phase metadata == "DRAFT"). Reads the proposed email body from the agent's
  response. Three checks: (1) body.contains("[unsubscribe]"), (2) prospectHqCountry is not in
  blocked-regions.json, (3) body.contains("[sender-address]"). Collects all failures.
  On any failure, returns Guardrail.reject("compliance-violation: " + failedRulesList). On
  accept, passes. Also calls ProspectEntity.recordComplianceCheck(ComplianceCheckResult{
  passed: true, failedRules: List.of(), checkedAt: now}) on success.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9969 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/prospects.jsonl with 5 seeded prospect lines covering the
  three companies named in the user flows plus two extras.

- src/main/resources/sample-data/prospects/*.json — three files keyed by company slug, each
  carrying FirmographicData with 3-5 IntentSignal entries with deterministic content so
  ResearchTools.lookupFirmographics returns the same data across restarts.

- src/main/resources/templates/email-templates.json — three named templates (
  "saas-technical-buyer", "saas-operations-buyer", "saas-founders") each with subject and
  body placeholders for {{companyName}}, {{personalizationHook}}, [unsubscribe],
  [sender-address].

- src/main/resources/compliance/blocked-regions.json — a list of ISO-3166-1 alpha-2 country
  codes that the sample treats as blocked (e.g. ["XX"]) for demo purposes. Deployers replace
  this with their real list.

- src/main/resources/sent-log/.gitkeep — empty file to create the directory; actual sent
  records appended at runtime.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (G1, H2, HITL1) matching the
  mechanisms in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.pii = true (contact emails are
  PII), decisions.authority_level = execute (the pipeline sends real email on approval),
  oversight.human_in_loop = true (HITL approval gate before every send), operations.agent_count
  = 1, operations.agent_pattern = sequential-pipeline, failure.failure_modes including
  "send-without-approval", "compliance-violation", "hallucinated-personalization",
  "reviewer-sla-exceeded"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/outreach-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: LeadPilot Lite - Personalized Cold
  Email", prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of prospect cards; right = selected-prospect detail with research summary, draft
  email, compliance badge, HITL approval panel, eval-score chip, rejection-log strip). Browser
  title exactly: <title>Akka Sample: LeadPilot Lite - Personalized Cold Email</title>. No
  subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build
        forwards the value from the Claude session env to the JVM via the MCP tool's
        environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml;
        /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using the
        matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(prospectId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    research-prospect.json — 5 ProspectProfile entries, each with FirmographicData and
      3-5 IntentSignal items. tool_calls array contains lookupFirmographics + fetchIntentSignals.
    draft-email.json — 5 EmailDraft entries paired one-to-one with the research entries,
      each with subject + body containing [unsubscribe] and [sender-address] sentinels.
      tool_calls contain personalizeLine + renderTemplate. Plus 1 entry whose body is MISSING
      the [unsubscribe] sentinel — ComplianceGuardrail fires, agent revises; J3 verifies this.
      Plus 1 entry that calls sendEmail during the DRAFT phase — SendGuardrail fires; J2
      verifies this.
    send-email.json — 5 OutreachEmail entries paired one-to-one with approved drafts,
      tool_calls contain sendEmail; each carries a SendReceipt with a deterministic messageId.
- A MockModelProvider.seedFor(prospectId) helper makes per-prospect selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. OutreachAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion OutreachTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (researchStep
  60s, draftStep 60s, approvalStep 86400s, sendStep 30s, error 5s).
- Lesson 6: every nullable lifecycle field on the ProspectRecord row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: OutreachTasks.java with RESEARCH_PROSPECT, DRAFT_EMAIL, SEND_EMAIL constants
  is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9969 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block. Without these, state names render
  black-on-black and arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (OutreachAgent). Both
  guardrails and the HITL gate are registered on or wired around the single agent.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent, but
  SendGuardrail is the runtime mechanism that enforces the send gate. Do NOT conditionally
  register tools per task.
- Task dependency is carried by typed task results: researchStep writes ProspectProfile onto
  the entity, draftStep reads it and builds the DRAFT task's instruction context from it,
  sendStep reads both. The agent itself is stateless across phases.
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
