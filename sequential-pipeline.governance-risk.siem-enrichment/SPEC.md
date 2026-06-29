# SPEC — siem-enrichment

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** SIEM Alert Enrichment with MITRE ATT\&CK and Zendesk.
**One-line pitch:** An operator submits a raw SIEM alert; one `AlertEnrichmentAgent` walks it through three task phases — **FETCH** raw alert detail, **ENRICH** it with MITRE ATT\&CK technique mappings, **TRIAGE** by opening a structured Zendesk ticket — with each phase gated on the prior phase's recorded output and the ticket-creation tool rejected until enrichment has been confirmed.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a governance-risk domain. One `AlertEnrichmentAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the FETCH task's typed output becomes the ENRICH task's instruction context; the ENRICH task's typed output becomes the TRIAGE task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Two governance mechanisms are wired around the pipeline:

- A **`before-tool-call` guardrail** sits between the agent and every tool call. It looks at the call's declared phase (`FETCH` / `ENRICH` / `TRIAGE`) and the current `AlertEntity` status. A TRIAGE-phase tool (`openZendeskTicket`) called while the entity has not yet recorded `EnrichmentProduced` is rejected before the tool body runs. The rejection returns a structured error to the agent so the task loop can correct course inside its iteration budget. This is the right enforcement point: the ticket-creation tool makes an external write; it must never fire on an unenriched alert.
- An **`on-incident-reporter`** runs immediately after `TicketCreated` lands, as `evalStep` inside the workflow. A deterministic, rule-based `TriageQualityScorer` (no LLM call — the eval is rule-based so the same enriched alert always scores the same) checks that every attack pattern has a MITRE technique ID, that every technique ID references a known ATT\&CK technique in the in-process registry, that the ticket's severity matches the highest-confidence attack pattern's impact level, and that the ticket's assigned-team field is non-empty.

The blueprint shows that a sequential pipeline is not just a chain of LLM calls — the task-boundary handoffs are the right cut to enforce both the dependency contract and the Zendesk write-scope isolation.

## 3. User-facing flows

The operator opens the App UI tab.

1. The operator picks an alert from the seeded list (or pastes a raw alert JSON string) and clicks **Run pipeline**. The UI POSTs to `/api/alerts` and receives an `alertId`.
2. The card appears in the live list in `RECEIVED` state. Within ~1 s it transitions to `FETCHING` — the workflow has started `fetchStep` and the agent has been handed the FETCH task.
3. Within ~10–20 s the card reaches `ENRICHING` — the typed `AlertDetail` is visible in the card detail (a small table of raw fields from the SIEM event). The agent's FETCH task returned; the workflow recorded `FetchCompleted` and ran the ENRICH task.
4. Within ~10–20 s more the card reaches `TRIAGING`. The `EnrichedAlert` is visible (attack-pattern list + technique mapping table).
5. Within ~10–20 s more the card reaches `EVALUATED`. The right pane now shows the full typed `TriageTicket` — title, severity, assigned team, MITRE technique list, Zendesk ticket ID — plus an eval score chip (1–5) and a one-line rationale.
6. The operator can submit another alert; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AlertEndpoint` | `HttpEndpoint` | `/api/alerts/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `AlertEntity`, `AlertView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `AlertEntity` | `EventSourcedEntity` | Per-alert lifecycle: received → fetching → fetched → enriching → enriched → triaging → triaged → evaluated. Source of truth. | `AlertEndpoint`, `AlertEnrichmentWorkflow` | `AlertView` |
| `AlertEnrichmentWorkflow` | `Workflow` | One workflow per alertId. Steps: `fetchStep` → `enrichStep` → `ticketStep` → `evalStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `AlertEndpoint` after `RECEIVED` | `AlertEnrichmentAgent`, `AlertEntity` |
| `AlertEnrichmentAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `AlertTasks.java`: `FETCH_ALERT` → `AlertDetail`, `ENRICH_ALERT` → `EnrichedAlert`, `CREATE_TICKET` → `TriageTicket`. Each task is registered with the phase-appropriate function tools. | invoked by `AlertEnrichmentWorkflow` | returns typed results |
| `FetchTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `fetchAlertDetail(alertId)` and `fetchHostContext(hostname)`. Reads from `src/main/resources/sample-data/alerts/*.json` for deterministic offline output. | called from FETCH task | returns `AlertDetail` |
| `EnrichTools` | function-tools class | Implements `lookupMitreTechnique(indicator)` and `mapAttackPattern(technique)`. Pure in-memory lookups against bundled ATT\&CK technique registry. | called from ENRICH task | returns `List<AttackPattern>` |
| `TicketTools` | function-tools class | Implements `openZendeskTicket(triageRequest)` and `assignTicketTeam(ticketId, team)`. In-process Zendesk adapter; returns a deterministic stub `ZendeskTicketRef`. | called from TRIAGE task | returns `TriageTicket` |
| `TicketScopeGuardrail` | `before-tool-call` guardrail (registered on `AlertEnrichmentAgent`) | Reads the in-flight task's declared phase and the current `AlertEntity` status. Rejects any TRIAGE-phase tool call whose enrichment precondition has not been satisfied. | every tool call on every task | accept / structured-reject |
| `TriageQualityScorer` | plain class (no Akka primitive) | Pure deterministic on-incident evaluator. Inputs: `TriageTicket`, `EnrichedAlert`, `AlertDetail`. Output: `EvalResult{score, rationale}`. | called from `evalStep` | returns score |
| `AlertView` | `View` | Read model: one row per alert for the UI. | `AlertEntity` events | `AlertEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record RawAlertField(String key, String value) {}

record AlertDetail(
    String alertId,
    String ruleName,
    String severity,           // "LOW" | "MEDIUM" | "HIGH" | "CRITICAL"
    String sourceIp,
    String destinationIp,
    String hostname,
    List<RawAlertField> rawFields,
    Instant detectedAt
) {}

record AttackPattern(
    String patternId,          // "ap-<8 hex>"
    String techniqueId,        // e.g. "T1059.001"
    String techniqueName,
    String tactic,             // e.g. "Execution"
    String confidence,         // "LOW" | "MEDIUM" | "HIGH"
    String indicator
) {}

record EnrichedAlert(
    AlertDetail alertDetail,
    List<AttackPattern> attackPatterns,
    String derivedSeverity,    // highest-confidence pattern impact
    Instant enrichedAt
) {}

record ZendeskTicketRef(
    String ticketId,
    String ticketUrl
) {}

record TriageTicket(
    String alertId,
    String title,
    String severity,
    String assignedTeam,
    List<AttackPattern> attackPatterns,
    ZendeskTicketRef zendeskRef,
    String summary,
    Instant createdAt
) {}

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record AlertRecord(
    String alertId,
    Optional<String> rawAlertJson,
    Optional<AlertDetail> alertDetail,
    Optional<EnrichedAlert> enrichedAlert,
    Optional<TriageTicket> triageTicket,
    Optional<EvalResult> eval,
    AlertStatus status,
    Instant receivedAt,
    Optional<Instant> finishedAt
) {}

enum AlertStatus {
    RECEIVED, FETCHING, FETCHED, ENRICHING, ENRICHED,
    TRIAGING, TRIAGED, EVALUATED, FAILED
}
```

Events on `AlertEntity`: `AlertReceived`, `FetchStarted`, `FetchCompleted`, `EnrichStarted`, `EnrichmentProduced`, `TriageStarted`, `TicketCreated`, `EvaluationScored`, `GuardrailRejected`, `AlertFailed`.

Every nullable lifecycle field on the `AlertRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/alerts` — body `{ rawAlertJson }` → `{ alertId }`.
- `GET /api/alerts` — list all alerts, newest-first.
- `GET /api/alerts/{id}` — one alert.
- `GET /api/alerts/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: SIEM Alert Enrichment</title>`.

The App UI tab is a two-column layout: a left rail with the live list of alerts (status pill + rule name + age) and a right pane with the selected alert's detail — raw alert fields table, attack-pattern mapping, triage ticket, eval score chip, and a guardrail-rejection log strip if any phase-gate rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-tool-call` guardrail (ticket-scope gate)**: `TicketScopeGuardrail` is registered on `AlertEnrichmentAgent` and runs before every tool call. It reads the in-flight `Task`'s declared phase (encoded as a constant on each function-tool class — `Phase.FETCH`, `Phase.ENRICH`, `Phase.TRIAGE`) and the current `AlertEntity.status` for the alert the task is bound to. The accept rule is precise: `FETCH` tools require `status ∈ {RECEIVED, FETCHING}`; `ENRICH` tools require `status ∈ {FETCHED, ENRICHING}` AND `alertDetail.isPresent()`; `TRIAGE` tools require `status ∈ {ENRICHED, TRIAGING}` AND `enrichedAlert.isPresent()`. On reject, the guardrail returns a structured `scope-violation` error to the agent loop and the workflow records a `GuardrailRejected{phase, tool, reason}` event for visibility. The agent loop retries within its 4-iteration budget. This gate is the critical external-write boundary: Zendesk tickets must never be opened for alerts that have not been enriched.
- **E1 — `on-incident-reporter`**: runs immediately after `TicketCreated` lands, as `evalStep` inside the workflow. `TriageQualityScorer` is a deterministic rule-based scorer (no LLM call — keeping the single-agent pipeline invariant honest): every attack pattern must have a non-empty `techniqueId` (ATT\&CK technique presence), every `techniqueId` must resolve against the bundled in-process ATT\&CK registry (technique validity), the ticket `severity` must match the derived severity from the highest-confidence attack pattern (severity accuracy), and the `assignedTeam` field must be non-empty (routing completeness). Emits `EvaluationScored{score:1..5, rationale}` on a one-point-per-rule basis with a clear sentence pinpointing the largest gap.

## 9. Agent prompts

- `AlertEnrichmentAgent` → `prompts/alert-enrichment-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Operator submits the seeded alert `lateral-movement-01`; within 60 s the alert reaches `EVALUATED` with non-empty attack patterns, ≥ 1 MITRE technique, a Zendesk ticket reference, and an eval score chip on the card.
2. **J2** — The agent's first iteration on an alert calls a TRIAGE-phase tool (`openZendeskTicket`) before `EnrichmentProduced` has been recorded (mock LLM path). `TicketScopeGuardrail` rejects the call; a `GuardrailRejected` event lands on the entity; the agent retries in-phase; the alert eventually completes correctly. The UI's rejection-log strip shows the one rejected call.
3. **J3** — An alert whose mock-LLM trajectory produces a triage ticket with an attack pattern `techniqueId` absent from the bundled ATT\&CK registry is scored 1 with a rationale naming the unrecognised technique; the UI flags the card.
4. **J4** — Each task's instructions, attachments, and tool calls are visible in the per-alert trace (logged at `INFO`); the FETCH task's log shows only FETCH-tool calls, the ENRICH task's log shows only ENRICH-tool calls, the TRIAGE task's log shows only TRIAGE-tool calls. The trace is empirical evidence the dependency contract holds end-to-end.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named siem-enrichment demonstrating the sequential-pipeline x governance-risk
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-governance-risk-siem-enrichment. Java package
io.akka.samples.siemalertenrichmentwithmitreattckandzendesk. Akka 3.6.0. HTTP port 9384.

Components to wire (exactly):

- 1 AutonomousAgent AlertEnrichmentAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/alert-enrichment-agent.md>) and three .capability(TaskAcceptance.of(TASK)
  .maxIterationsPerTask(4)) entries — one per declared Task. Function tools are registered
  with .tools(...) — the FETCH, ENRICH, and TRIAGE tool sets are ALL registered on the agent;
  phase gating is the job of TicketScopeGuardrail, NOT of conditional .tools(...) wiring. The
  before-tool-call guardrail (TicketScopeGuardrail) is registered on the agent via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries within its
  4-iteration budget.

- 1 Workflow AlertEnrichmentWorkflow per alertId with four steps:
  * fetchStep — emits FetchStarted on the entity, then calls componentClient
    .forAutonomousAgent(AlertEnrichmentAgent.class, "agent-" + alertId).runSingleTask(
      TaskDef.instructions("AlertId: " + alertId + "\nPhase: FETCH\nUse the fetch tools to
      retrieve the alert detail and host context for this alert.")
        .metadata("alertId", alertId)
        .metadata("phase", "FETCH")
        .taskType(AlertTasks.FETCH_ALERT)
    ). Reads forTask(taskId).result(FETCH_ALERT) to get AlertDetail. Writes
    AlertEntity.recordAlertDetail(alertDetail). WorkflowSettings.stepTimeout 60s.
  * enrichStep — emits EnrichStarted, then runSingleTask with TaskDef.instructions
    (formatEnrichContext(alertDetail)) and metadata.phase = "ENRICH", taskType
    ENRICH_ALERT. Writes AlertEntity.recordEnrichment(enrichedAlert). stepTimeout 60s.
  * ticketStep — emits TriageStarted, then runSingleTask with TaskDef.instructions
    (formatTriageContext(enrichedAlert, alertDetail)) and metadata.phase = "TRIAGE", taskType
    CREATE_TICKET. Writes AlertEntity.recordTriageTicket(triageTicket). stepTimeout 60s.
  * evalStep — runs the deterministic TriageQualityScorer over (triageTicket, enrichedAlert,
    alertDetail) and writes AlertEntity.recordEvaluation(eval). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(AlertEnrichmentWorkflow::error). The error step writes
  AlertFailed and ends.

- 1 EventSourcedEntity AlertEntity (one per alertId). State AlertRecord{alertId,
  rawAlertJson: Optional<String>, alertDetail: Optional<AlertDetail>,
  enrichedAlert: Optional<EnrichedAlert>, triageTicket: Optional<TriageTicket>,
  eval: Optional<EvalResult>, status: AlertStatus, receivedAt: Instant,
  finishedAt: Optional<Instant>}. AlertStatus enum: RECEIVED, FETCHING, FETCHED, ENRICHING,
  ENRICHED, TRIAGING, TRIAGED, EVALUATED, FAILED. Events: AlertReceived{rawAlertJson},
  FetchStarted, FetchCompleted{alertDetail}, EnrichStarted, EnrichmentProduced{enrichedAlert},
  TriageStarted, TicketCreated{triageTicket}, EvaluationScored{eval},
  GuardrailRejected{phase, tool, reason}, AlertFailed{reason}.
  Commands: receive, startFetch, recordAlertDetail, startEnrich, recordEnrichment, startTriage,
  recordTriageTicket, recordEvaluation, recordGuardrailRejection, fail, getAlert. emptyState()
  returns AlertRecord.initial("") with all Optional fields as Optional.empty() and no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 View AlertView with row type AlertRow that mirrors AlertRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes AlertEntity events. ONE
  query getAllAlerts: SELECT * AS alerts FROM alert_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * AlertEndpoint at /api with POST /alerts (body {rawAlertJson}; mints alertId; calls
    AlertEntity.receive(rawAlertJson); then starts AlertEnrichmentWorkflow with id
    "pipeline-" + alertId; returns {alertId}), GET /alerts (list from
    getAllAlerts, sorted newest-first), GET /alerts/{id} (one row), GET
    /alerts/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- AlertTasks.java declaring three Task<R> constants:
    FETCH_ALERT = Task.name("Fetch alert").description("Retrieve the raw alert detail and
      host context using fetchAlertDetail and fetchHostContext").resultConformsTo(AlertDetail.class);
    ENRICH_ALERT = Task.name("Enrich alert").description("Map alert indicators to MITRE ATT&CK
      techniques using lookupMitreTechnique and mapAttackPattern")
      .resultConformsTo(EnrichedAlert.class);
    CREATE_TICKET = Task.name("Create ticket").description("Open a Zendesk triage ticket for
      the enriched alert using openZendeskTicket and assignTicketTeam")
      .resultConformsTo(TriageTicket.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Phase.java — enum {FETCH, ENRICH, TRIAGE}. Each function-tool method is annotated with
  the constant phase.

- FetchTools.java — @FunctionTool fetchAlertDetail(String alertId) -> AlertDetail reading
  from src/main/resources/sample-data/alerts/*.json keyed by alertId; @FunctionTool
  fetchHostContext(String hostname) -> String reading the matching alert's host-context entry.

- EnrichTools.java — @FunctionTool lookupMitreTechnique(String indicator) -> List<String>
  (returns matching techniqueIds from in-process ATT&CK registry keyed by indicator);
  @FunctionTool mapAttackPattern(String techniqueId) -> AttackPattern (looks up the full
  technique record and mints a patternId "ap-" + sha1(techniqueId).substring(0,8)).

- TicketTools.java — @FunctionTool openZendeskTicket(TriageTicketRequest request) ->
  ZendeskTicketRef (in-process stub returning deterministic ticket id and url);
  @FunctionTool assignTicketTeam(String ticketId, String team) -> String (updates the stub
  assignment and returns "assigned").

- TicketScopeGuardrail.java — implements the before-tool-call hook. Reads the candidate tool
  call's @FunctionTool.phase attribute, looks up the AlertEntity status by alertId
  (carried in the TaskDef metadata), applies the accept matrix from Section 8, and either
  passes or returns Guardrail.reject("scope-violation: <tool> requires <precondition>, saw
  <status>"). On reject ALSO calls AlertEntity.recordGuardrailRejection(phase, tool,
  reason) so the rejection is visible in the UI's rejection-log strip and in the audit log.

- TriageQualityScorer.java — pure deterministic logic (no LLM). Inputs: TriageTicket,
  EnrichedAlert, AlertDetail. Outputs: EvalResult with score and rationale. Four checks,
  one point per check satisfied, starting from a base of 1: ATT&CK technique presence
  (every AttackPattern has a non-empty techniqueId), technique validity (every techniqueId
  resolves in the bundled ATT&CK registry), severity accuracy (ticket.severity matches
  the derivedSeverity from EnrichedAlert), and routing completeness (assignedTeam is
  non-empty). Score range 1-5. Rationale names the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9384 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/alerts.jsonl with 5 seeded alert lines covering different
  attack scenarios (lateral movement, privilege escalation, data exfiltration, C2 beaconing,
  credential dumping).

- src/main/resources/sample-data/alerts/*.json — five files keyed by seeded alertId, each
  carrying an AlertDetail with realistic SIEM fields and matching ATT&CK technique indicators
  so EnrichTools.lookupMitreTechnique returns consistent results across restarts.

- src/main/resources/sample-data/mitre-techniques.json — a trimmed ATT&CK technique registry
  covering the techniques referenced in the seeded alerts, sufficient for TriageQualityScorer
  to validate every sample ticket end-to-end.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, E1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = true (SIEM alerts
  contain IP addresses and hostnames), decisions.authority_level = recommend-only (the ticket
  is advisory; the SOC analyst acts on it), oversight.human_in_loop = true (a SOC analyst
  reviews the ticket before closing), operations.agent_count = 1, operations.agent_pattern =
  sequential-pipeline, failure.failure_modes including "unenriched-ticket",
  "unrecognised-mitre-technique", "scope-violation", "severity-mismatch",
  "unrouted-ticket"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/alert-enrichment-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: SIEM Alert Enrichment with MITRE ATT&CK
  and Zendesk", prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration section. NO
  governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of alert cards; right = selected-alert detail with rule-name header, raw-fields
  table, attack-pattern mapping, triage ticket, eval-score chip, rejection-log strip).
  Browser title exactly: <title>Akka Sample: SIEM Alert Enrichment</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java per-task dispatch on Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json.
- Per-task mock-response shapes:
    fetch-alert.json — 5 AlertDetail entries, one per seeded alertId, each with realistic
      SIEM fields, tool_calls containing fetchAlertDetail + fetchHostContext.
    enrich-alert.json — 5 EnrichedAlert entries, each with 2-3 AttackPattern items,
      tool_calls containing lookupMitreTechnique + mapAttackPattern. Plus 1 deliberately
      PHASE-VIOLATING entry whose tool_calls array starts with openZendeskTicket (a TRIAGE-
      phase tool called during ENRICH) — the guardrail rejects it on the first iteration of
      every 3rd alert (modulo seed).
    create-ticket.json — 5 TriageTicket entries, each referencing a valid Zendesk stub.
      Plus 1 deliberately INVALID-TECHNIQUE entry whose first AttackPattern carries a
      techniqueId absent from the bundled registry — the workflow's evalStep scores it 1.
- MockModelProvider.seedFor(alertId) makes per-alert selection deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. AlertEnrichmentAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion AlertTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (fetchStep
  60s, enrichStep 60s, ticketStep 60s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the AlertRecord row record is Optional<T>.
- Lesson 7: AlertTasks.java with FETCH_ALERT, ENRICH_ALERT, CREATE_TICKET constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9384 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: exactly ONE AutonomousAgent (AlertEnrichmentAgent). The
  on-incident eval is rule-based (TriageQualityScorer.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent, but
  TicketScopeGuardrail is the runtime mechanism that enforces the phase order. Do NOT
  conditionally register tools per task.
- Task dependency is carried by typed task results: fetchStep writes AlertDetail onto the
  entity, enrichStep reads it and builds the ENRICH task's instruction context from it,
  ticketStep reads both. The agent itself is stateless across phases.
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
