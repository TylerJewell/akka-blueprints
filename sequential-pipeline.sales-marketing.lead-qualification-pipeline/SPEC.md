# SPEC — inbound-lead-qualification

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Inbound Lead Qualification & Research Agent.
**One-line pitch:** A lead form submission triggers one `LeadAgent` walking the record through three task phases — **ENRICH** firmographic data, **QUALIFY** the lead against a scoring rubric, **NOTIFY** the assigned sales rep via Slack — with each phase gated on the prior phase's recorded output, PII sanitized before enrichment tools run, and every qualification score sampled for accuracy.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a sales-marketing domain. One `LeadAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the ENRICH task's typed output becomes the QUALIFY task's instruction context; the QUALIFY task's typed output becomes the NOTIFY task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Three governance mechanisms are wired around the pipeline:

- A **`before-tool-call` guardrail** (`SlackGuardrail`) sits between the agent and the `postLeadToSlack` tool. It reads the current `LeadEntity` status and rejects the Slack write until `QualificationScored` has been recorded. This prevents an unqualified or partially-enriched lead from triggering a sales notification. On reject, the guardrail returns a structured error to the agent loop and the workflow records a `GuardrailRejected{phase, tool, reason}` event for visibility.

- A **PII sanitizer** (`PiiSanitizer`) runs at the system level before the ENRICH task's instruction context is handed to the agent. It replaces the submitter's raw name, email address, and personal company contact with pseudonymous tokens, so those values never appear in an LLM call or in an enrichment tool's outbound request. The original values are preserved on `LeadEntity` for CRM handoff but are never forwarded downstream.

- An **`on-decision-eval`** (`QualificationEvaluator`) runs immediately after `QualificationScored` lands, as `evalStep` inside the workflow. A deterministic, rule-based scorer checks that the stated qualification tier is consistent with the enrichment evidence: employee count supports the tier band, revenue estimate supports the tier band, and the stated use-case matches at least one known buying pattern for the tier. Emits `EvaluationRecorded{confidence: HIGH/MED/LOW, rationale}` for each qualified lead.

The blueprint shows that the Slack write is an external side-effect that belongs at the end of the pipeline — the guardrail is the right mechanism to enforce that ordering, and the evaluator is the right mechanism to surface qualification drift over time.

## 3. User-facing flows

The user opens the App UI tab.

1. The user fills in the **Submit Lead** form: first name, last name, email, company name, company size (dropdown: 1–10 / 11–50 / 51–200 / 201–1 000 / 1 001+), message (optional). Or picks one of three seeded leads — `Acme Corp / VP Engineering / 450 employees`, `Bright Futures / Head of Product / 22 employees`, `Quantum Systems / CTO / 3 200 employees`.
2. The user clicks **Submit lead**. The UI POSTs to `/api/leads` and receives a `leadId`.
3. The card appears in the live list in `SUBMITTED` state. Within ~1 s it transitions to `ENRICHING` — the workflow has started `enrichStep` and the agent has been handed the ENRICH task.
4. Within ~10–20 s the card reaches `ENRICHED` — the `FirmographicProfile` is visible in the card detail (industry, estimated ARR, employee band, tech stack signals, LinkedIn company page). The PII sanitizer has already run; the agent never saw the raw email or name.
5. Within ~10–20 s more the card reaches `QUALIFIED`. The `QualificationScore` is visible (tier: HOT / WARM / COLD, score 0–100, rationale, assigned rep name).
6. Within ~10–20 s more the card reaches `NOTIFIED`. The right pane shows the Slack notification payload (channel, text, blocks). The `SlackGuardrail` accepted the `postLeadToSlack` call because `QualificationScored` was present.
7. Within ~1 s the card reaches `EVALUATED`. The right pane shows the confidence chip (HIGH / MED / LOW) and a one-line rationale. The card border highlights amber if confidence is LOW.
8. The user can submit another lead; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `LeadEndpoint` | `HttpEndpoint` | `/api/leads/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `LeadEntity`, `LeadView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `LeadEntity` | `EventSourcedEntity` | Per-lead lifecycle: submitted → enriching → enriched → qualifying → qualified → notifying → notified → evaluated. Source of truth. | `LeadEndpoint`, `LeadPipelineWorkflow` | `LeadView` |
| `LeadPipelineWorkflow` | `Workflow` | One workflow per lead. Steps: `enrichStep` → `qualifyStep` → `notifyStep` → `evalStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `LeadEndpoint` after `SUBMITTED` | `LeadAgent`, `LeadEntity` |
| `LeadAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `LeadTasks.java`: `ENRICH_LEAD` → `FirmographicProfile`, `QUALIFY_LEAD` → `QualificationScore`, `NOTIFY_SALES` → `SlackPayload`. Each task is registered with the phase-appropriate function tools. | invoked by `LeadPipelineWorkflow` | returns typed results |
| `EnrichTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `lookupFirmographics(domain)` and `fetchTechStack(domain)`. Reads from `src/main/resources/sample-data/firmographics/*.json` for deterministic offline output. | called from ENRICH task | returns `FirmographicProfile` |
| `QualifyTools` | function-tools class | Implements `scoreByRubric(profile, formData)` and `assignRep(score)`. Rule-based scoring against a tiered rubric. | called from QUALIFY task | returns `QualificationScore` |
| `NotifyTools` | function-tools class | Implements `buildSlackMessage(score, profile)` and `postLeadToSlack(channel, payload)`. `postLeadToSlack` is the external Slack write. | called from NOTIFY task | returns `SlackPayload` |
| `PiiSanitizer` | system-level sanitizer (registered on `LeadAgent`) | Replaces raw name, email, and personal contact with pseudonymous tokens before the ENRICH task's instructions are forwarded to the LLM or any tool. | ENRICH task instruction context | anonymized instruction string |
| `SlackGuardrail` | `before-tool-call` guardrail (registered on `LeadAgent`) | Reads the current `LeadEntity.status` for the lead the task is bound to. Rejects `postLeadToSlack` unless `status ∈ {QUALIFIED, NOTIFYING}` AND `score.isPresent()`. | every tool call on NOTIFY task | accept / structured-reject |
| `QualificationEvaluator` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `QualificationScore`, `FirmographicProfile`, `LeadFormData`. Output: `EvalRecord{confidence, rationale}`. | called from `evalStep` | returns confidence |
| `LeadView` | `View` | Read model: one row per lead for the UI. | `LeadEntity` events | `LeadEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record LeadFormData(
    String leadId,
    String firstName,
    String lastName,
    String email,
    String companyName,
    CompanySize companySize,
    Optional<String> message,
    Instant submittedAt
) {}

enum CompanySize { XS, S, M, L, XL }  // 1-10, 11-50, 51-200, 201-1000, 1001+

record TechStackSignal(String technology, String evidence) {}

record FirmographicProfile(
    String domain,
    String industry,
    String estimatedArrBand,   // e.g. "$1M–$5M"
    String employeeBand,       // e.g. "201–1000"
    List<TechStackSignal> techStack,
    Optional<String> linkedinUrl,
    Instant enrichedAt
) {}

record QualificationScore(
    String leadId,
    LeadTier tier,             // HOT, WARM, COLD
    int score,                 // 0–100
    String rationale,
    String assignedRepName,
    String assignedRepSlackId,
    Instant qualifiedAt
) {}

enum LeadTier { HOT, WARM, COLD }

record SlackBlock(String type, String text) {}

record SlackPayload(
    String channel,
    String text,
    List<SlackBlock> blocks,
    Instant sentAt
) {}

record EvalRecord(
    EvalConfidence confidence,  // HIGH, MED, LOW
    String rationale,
    Instant evaluatedAt
) {}

enum EvalConfidence { HIGH, MED, LOW }

record LeadRecord(
    String leadId,
    Optional<LeadFormData> formData,
    Optional<FirmographicProfile> profile,
    Optional<QualificationScore> score,
    Optional<SlackPayload> notification,
    Optional<EvalRecord> eval,
    LeadStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum LeadStatus {
    SUBMITTED, ENRICHING, ENRICHED, QUALIFYING, QUALIFIED,
    NOTIFYING, NOTIFIED, EVALUATED, FAILED
}
```

Events on `LeadEntity`: `LeadSubmitted`, `EnrichStarted`, `EnrichmentCompleted`, `QualifyStarted`, `QualificationScored`, `NotifyStarted`, `NotificationSent`, `EvaluationRecorded`, `GuardrailRejected`, `LeadFailed`.

Every nullable lifecycle field on the `LeadRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/leads` — body `LeadFormData` → `{ leadId }`.
- `GET /api/leads` — list all leads, newest-first.
- `GET /api/leads/{id}` — one lead.
- `GET /api/leads/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Inbound Lead Qualification & Research Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of leads (status pill + company name + tier chip + age) and a right pane with the selected lead's detail — company, enrichment profile, qualification score, Slack notification payload, eval confidence chip, and a guardrail-rejection log strip if any phase-gate rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-tool-call` guardrail (Slack write gate)**: `SlackGuardrail` is registered on `LeadAgent` and runs before every tool call on the NOTIFY task. It reads the current `LeadEntity.status` for the lead keyed by `leadId` in the task's metadata. The accept rule: `postLeadToSlack` requires `status ∈ {QUALIFIED, NOTIFYING}` AND `score.isPresent()`. On reject, the guardrail returns a structured `slack-write-blocked` error to the agent loop and the workflow records a `GuardrailRejected{phase: "NOTIFY", tool: "postLeadToSlack", reason}` event. The agent loop retries within its 4-iteration budget. All other NOTIFY-phase tools (`buildSlackMessage`) are allowed regardless of status.

- **S1 — PII sanitizer (lead data)**: `PiiSanitizer` is registered on `LeadAgent` as a system-level sanitizer. It runs on the ENRICH task's instruction string before it is forwarded to the LLM or any tool. The sanitizer replaces `firstName`, `lastName`, `email`, and the personal company contact details with stable pseudonymous tokens (`[NAME-1]`, `[EMAIL-1]`, etc.) derived from a session-scoped salt. The original values are retained on `LeadEntity` for CRM handoff and are never re-introduced into agent context after sanitization.

- **E1 — `on-decision-eval` (qualification accuracy sampling)**: `QualificationEvaluator` runs inside `LeadPipelineWorkflow.evalStep`, fired immediately after `QualificationScored` lands. The evaluator is deterministic and rule-based (no LLM call). Three checks, one point per check satisfied, on a base of 1: (1) tier-by-size consistency — `LeadTier.HOT` requires `employeeBand ∈ {201–1 000, 1 001+}` or `estimatedArrBand ≥ $5 M`; (2) rep assignment completeness — `assignedRepSlackId` must be non-blank; (3) score-tier alignment — `score ≥ 70` requires `tier = HOT`, `score 40–69` requires `tier = WARM`, `score < 40` requires `tier = COLD`. Emits `EvaluationRecorded{confidence: HIGH (3 pts) / MED (2 pts) / LOW (1 pt), rationale}`. LOW-confidence leads are flagged on the UI card for rep review.

## 9. Agent prompts

- `LeadAgent` → `prompts/lead-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Submit the seeded lead `Quantum Systems / CTO / 3 200 employees`; within 60 s the lead reaches `EVALUATED` with a non-empty `FirmographicProfile`, a `QualificationScore` with `tier = HOT`, a `SlackPayload` with non-empty `blocks`, and a confidence chip of HIGH.
2. **J2** — The agent's first iteration on a lead calls `postLeadToSlack` before `QualificationScored` has been recorded (mock LLM path). `SlackGuardrail` rejects the call; a `GuardrailRejected` event lands on the entity; the agent retries in-phase; the lead eventually reaches `NOTIFIED`. The UI's rejection-log strip shows the one rejected call.
3. **J3** — A lead submitted with a 10-person company (`companySize = XS`) whose mock-LLM trajectory assigns `tier = HOT` and `score = 85` is evaluated with `confidence = LOW` (tier-by-size check fails); the UI flags the card amber.
4. **J4** — A submitted lead's enrichment-task log shows no raw email or name strings; the PII sanitizer's pseudonymous tokens appear instead. The original values are present on `LeadEntity` via `GET /api/leads/{id}`.
5. **J5** — Each task's tool calls are isolated: the ENRICH task's log shows only `lookupFirmographics` and `fetchTechStack`; the QUALIFY task's log shows only `scoreByRubric` and `assignRep`; the NOTIFY task's log shows only `buildSlackMessage` and `postLeadToSlack`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named inbound-lead-qualification demonstrating the sequential-pipeline x
sales-marketing cell. Requires a Slack bot token (SLACK_BOT_TOKEN env var) for the NOTIFY
phase; falls back to a mock Slack client that logs the payload locally when the token is
absent. Maven group io.akka.samples. Maven artifact
sequential-pipeline-sales-marketing-lead-qualification-pipeline. Java package
io.akka.samples.inboundleadqualificationresearchagent. Akka 3.6.0. HTTP port 9310.

Components to wire (exactly):

- 1 AutonomousAgent LeadAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/lead-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — ALL
  three tool sets (EnrichTools, QualifyTools, NotifyTools) are registered on the agent; phase
  gating is the job of SlackGuardrail and task metadata, NOT of conditional .tools(...) wiring.
  The before-tool-call guardrail (SlackGuardrail) and the PII sanitizer (PiiSanitizer) are both
  registered on the agent via the agent's guardrail-configuration block. On guardrail rejection
  the agent loop retries within its 4-iteration budget.

- 1 Workflow LeadPipelineWorkflow per leadId with four steps:
  * enrichStep — emits EnrichStarted on the entity, then calls componentClient
    .forAutonomousAgent(LeadAgent.class, "agent-" + leadId).runSingleTask(
      TaskDef.instructions("Lead domain: " + domain + "\nPhase: ENRICH\nUse lookupFirmographics
      and fetchTechStack to build the firmographic profile for this domain.")
        .metadata("leadId", leadId)
        .metadata("phase", "ENRICH")
        .taskType(LeadTasks.ENRICH_LEAD)
    ). Reads forTask(taskId).result(ENRICH_LEAD) to get FirmographicProfile. Writes
    LeadEntity.recordProfile(profile). WorkflowSettings.stepTimeout 60s.
  * qualifyStep — emits QualifyStarted, then runSingleTask with TaskDef.instructions
    (formatQualifyContext(profile, formData)) and metadata.phase = "QUALIFY", taskType
    QUALIFY_LEAD. Writes LeadEntity.recordScore(score). stepTimeout 60s.
  * notifyStep — emits NotifyStarted, then runSingleTask with TaskDef.instructions
    (formatNotifyContext(score, profile)) and metadata.phase = "NOTIFY", taskType
    NOTIFY_SALES. Writes LeadEntity.recordNotification(notification). stepTimeout 60s.
  * evalStep — runs the deterministic QualificationEvaluator over (score, profile, formData)
    and writes LeadEntity.recordEval(eval). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(LeadPipelineWorkflow::error). The error step writes
  LeadFailed and ends.

- 1 EventSourcedEntity LeadEntity (one per leadId). State LeadRecord{leadId,
  formData: Optional<LeadFormData>, profile: Optional<FirmographicProfile>,
  score: Optional<QualificationScore>, notification: Optional<SlackPayload>,
  eval: Optional<EvalRecord>, status: LeadStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. LeadStatus enum: SUBMITTED, ENRICHING, ENRICHED,
  QUALIFYING, QUALIFIED, NOTIFYING, NOTIFIED, EVALUATED, FAILED. Events:
  LeadSubmitted{formData}, EnrichStarted, EnrichmentCompleted{profile}, QualifyStarted,
  QualificationScored{score}, NotifyStarted, NotificationSent{notification},
  EvaluationRecorded{eval}, GuardrailRejected{phase, tool, reason}, LeadFailed{reason}.
  Commands: submit, startEnrich, recordProfile, startQualify, recordScore, startNotify,
  recordNotification, recordEval, recordGuardrailRejection, fail, getLead. emptyState()
  returns LeadRecord.initial("") with all Optional fields as Optional.empty() and no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 View LeadView with row type LeadRow that mirrors LeadRecord exactly (all Optional<T>
  lifecycle fields preserved). Table updater consumes LeadEntity events. ONE query
  getAllLeads: SELECT * AS leads FROM lead_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * LeadEndpoint at /api with POST /leads (body LeadFormData; mints leadId; calls
    LeadEntity.submit(formData); then starts LeadPipelineWorkflow with id
    "pipeline-" + leadId; returns {leadId}), GET /leads (list from getAllLeads,
    sorted newest-first), GET /leads/{id} (one row), GET /leads/sse (Server-Sent
    Events forwarded from the view's stream-updates), and three /api/metadata/*
    endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- LeadTasks.java declaring three Task<R> constants:
    ENRICH_LEAD = Task.name("Enrich lead").description("Look up firmographic data and
      tech-stack signals for the submitted company domain").resultConformsTo(
      FirmographicProfile.class);
    QUALIFY_LEAD = Task.name("Qualify lead").description("Score the enriched profile
      against the qualification rubric and assign a sales rep").resultConformsTo(
      QualificationScore.class);
    NOTIFY_SALES = Task.name("Notify sales").description("Build a Slack message block
      and post it to the assigned rep's channel").resultConformsTo(SlackPayload.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- LeadPhase.java — enum {ENRICH, QUALIFY, NOTIFY}. Each function-tool method is annotated
  with the constant phase (use a custom annotation if the SDK's @FunctionTool does not carry
  a phase field — the guardrail reads it from a parallel registry built at startup if so).

- EnrichTools.java — @FunctionTool lookupFirmographics(String domain) -> FirmographicProfile
  reading from src/main/resources/sample-data/firmographics/<domain>.json; @FunctionTool
  fetchTechStack(String domain) -> List<TechStackSignal> reading from the matching
  firmographic entry's techStack array.

- QualifyTools.java — @FunctionTool scoreByRubric(FirmographicProfile, LeadFormData) ->
  QualificationScore (deterministic tiering: XL/L → HOT if ARR ≥ $5M, WARM otherwise; M →
  WARM; S/XS → COLD; score 0-100 derived from band weightings); @FunctionTool
  assignRep(QualificationScore) -> String (round-robin rep assignment from
  src/main/resources/sample-data/reps.json).

- NotifyTools.java — @FunctionTool buildSlackMessage(QualificationScore,
  FirmographicProfile) -> SlackPayload (constructs blocks from score + profile fields);
  @FunctionTool postLeadToSlack(String channel, SlackPayload) -> SlackPayload (calls
  Slack API using SLACK_BOT_TOKEN env var; falls back to MockSlackClient when token absent).

- PiiSanitizer.java — implements the system-level sanitizer hook. Runs on the ENRICH task's
  instruction string. Replaces all occurrences of firstName, lastName, email, and personal
  company contact fields with stable pseudonymous tokens derived from sha1(leadId + salt).
  Returns the sanitized instruction string. The original field values on LeadEntity are never
  modified.

- SlackGuardrail.java — implements the before-tool-call hook. Checks whether the candidate
  tool call is postLeadToSlack. If so, looks up LeadEntity.status by leadId (carried in
  TaskDef metadata). Accepts only if status ∈ {QUALIFIED, NOTIFYING} AND score.isPresent().
  On reject ALSO calls LeadEntity.recordGuardrailRejection(phase, tool, reason) so the
  rejection is visible in the UI's rejection-log strip and in the audit log. On accept, the
  call proceeds. All other tool calls pass through unconditionally.

- QualificationEvaluator.java — pure deterministic logic (no LLM). Inputs: QualificationScore,
  FirmographicProfile, LeadFormData. Outputs: EvalRecord with confidence and rationale. Three
  checks: (1) tier-by-size consistency — HOT requires employeeBand ∈ {201-1000, 1001+} or
  estimatedArrBand ≥ $5M; (2) rep assignment completeness — assignedRepSlackId non-blank;
  (3) score-tier alignment — score ≥ 70 → HOT, 40–69 → WARM, < 40 → COLD. Confidence HIGH
  (3 checks pass), MED (2), LOW (1 or 0). Rationale names the failing check.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9310 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. Also: slack.bot-token = ${?SLACK_BOT_TOKEN}.

- src/main/resources/sample-events/leads.jsonl with 5 seeded lead lines covering the three
  surfaces named in J1-J5 plus two extras.

- src/main/resources/sample-data/firmographics/*.json — three files keyed by company domain,
  each carrying a full FirmographicProfile with deterministic content so EnrichTools returns
  the same profile across restarts.

- src/main/resources/sample-data/reps.json — list of 3 sales reps with name, email, and
  slackId fields for round-robin assignment.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (G1, S1, E1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors — general
  sales domain.

- risk-survey.yaml at the project root with data.data_classes.pii = true (lead name + email
  are PII), decisions.authority_level = recommend-only (the qualification score is advisory),
  oversight.human_in_loop = true (sales rep reads every score before acting), operations
  .agent_count = 1, operations.agent_pattern = sequential-pipeline, failure.failure_modes
  including "unqualified-lead-notified", "pii-leak-to-enrichment-tool",
  "score-tier-mismatch", "slack-write-before-qualification"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/lead-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Inbound Lead Qualification &
  Research Agent", prerequisites (including Slack credential sourcing), generate-the-system,
  what-you-get, customise-before-generating, what-gets-validated, license. NO Configuration
  section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of lead cards; right = selected-lead detail with company header, enrichment
  profile table, qualification score block, Slack payload preview, eval-confidence chip,
  rejection-log strip). Browser title exactly:
  <title>Akka Sample: Inbound Lead Qualification & Research Agent</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- Also ask separately whether SLACK_BOT_TOKEN is available, using the same five options.
  If absent, MockSlackClient is generated and the real postLeadToSlack is stubbed.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(leadId)), and
  deserialises into the task's typed return. Within a single task run, the mock drives
  tool-call sequences — each entry carries a "tool_calls" array the mock replays in order.
- Per-task mock-response shapes for THIS blueprint:
    enrich-lead.json — 5 FirmographicProfile entries keyed to seeded company domains, each
      with tool_calls of [lookupFirmographics, fetchTechStack].
    qualify-lead.json — 5 QualificationScore entries covering HOT/WARM/COLD tiers; includes
      1 deliberately MISALIGNED entry (score = 80, tier = COLD) so J3 fires on modulo seed.
    notify-sales.json — 5 SlackPayload entries; includes 1 entry whose tool_calls array
      starts with postLeadToSlack BEFORE buildSlackMessage — the SlackGuardrail rejects it
      on the first iteration of every 3rd lead (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(leadId) helper makes per-lead selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. LeadAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion LeadTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (enrichStep
  60s, qualifyStep 60s, notifyStep 60s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the LeadRecord row record is Optional<T>.
- Lesson 7: LeadTasks.java with ENRICH_LEAD, QUALIFY_LEAD, NOTIFY_SALES constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9310 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Requires Slack Adapter" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (LeadAgent).
  QualificationEvaluator does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent, but
  SlackGuardrail is the runtime mechanism that enforces the Slack write ordering. Do NOT
  conditionally register tools per task.
- Task dependency is carried by typed task results: enrichStep writes FirmographicProfile
  onto the entity, qualifyStep reads it and builds the QUALIFY task's context from it,
  notifyStep reads both. The agent itself is stateless across phases.
- The Overview tab's Try-it card shows just "/akka:build". Per Lesson 25, /akka:specify
  handles the key during generation.
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
