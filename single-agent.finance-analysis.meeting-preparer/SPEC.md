# SPEC — meeting-preparer

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** MeetingPreparer.
**One-line pitch:** A user submits an upcoming meeting (counterparty name, date/time, attendees, agenda topics); one AI agent pulls CRM history, financial highlights, and recent news, sanitizes the contact data, and returns a structured `MeetingBrief` — executive summary, key talking points, risk flags, financial highlights, and recent news.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the finance-analysis domain. One `BriefingAgent` (AutonomousAgent) produces the entire brief; the surrounding components only prepare its input and audit its output. One governance mechanism is wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw meeting request (which carries CRM contact data with personal identifiers) and the agent call — so the model never sees raw emails, phone numbers, or individual contact names from the CRM payload.

The blueprint shows that a single-agent pipeline can govern its data pipeline before the LLM call arrives. The on-decision evaluator scores each brief for completeness and actionability without making an LLM call, keeping the single-agent invariant honest.

## 3. User-facing flows

The user opens the App UI tab.

1. The user fills in the **Meeting request** form: counterparty name (e.g., "Meridian Capital Advisors"), meeting date/time, agenda topics (comma-separated), and their own name.
2. The user clicks **Prepare brief**. The UI POSTs to `/api/briefs` and receives a `briefId`.
3. The card appears in the live list in `REQUESTED` state. Within ~1 s, it transitions to `DATA_SANITIZED` — the redacted CRM payload is visible in the card detail, with a small list of PII categories the sanitizer found.
4. Within ~10–30 s, the workflow's `briefStep` completes. The card transitions to `BRIEFING` then `BRIEF_READY`. The brief appears: an executive summary paragraph, a numbered talking-points list, a risk-flags section (amber chips), a financial-highlights row, and a recent-news list.
5. Within ~1 s of the brief landing, the `evalStep` finishes. The card shows an **eval score** chip (1–5) plus a one-line rationale describing whether the brief's talking points are backed by evidence.
6. The user can submit another meeting request; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `BriefingEndpoint` | `HttpEndpoint` | `/api/briefs/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `BriefingEntity`, `BriefingView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `BriefingEntity` | `EventSourcedEntity` | Per-brief lifecycle: requested → data-sanitized → briefing → brief-ready → evaluated. Source of truth. | `BriefingEndpoint`, `ContactSanitizer`, `BriefingWorkflow` | `BriefingView` |
| `ContactSanitizer` | `Consumer` | Subscribes to `MeetingRequested` events; redacts PII from CRM contact payload; calls `BriefingEntity.attachSanitizedData`. | `BriefingEntity` events | `BriefingEntity` |
| `BriefingWorkflow` | `Workflow` | One workflow per brief. Steps: `awaitSanitizedStep` → `briefStep` → `evalStep`. | started by `ContactSanitizer` once sanitized event lands | `BriefingAgent`, `BriefingEntity` |
| `BriefingAgent` | `AutonomousAgent` | The one decision-making LLM. Receives meeting context in the task definition and the sanitized CRM snapshot as a task attachment; returns `MeetingBrief`. | invoked by `BriefingWorkflow` | returns brief |
| `BriefingView` | `View` | Read model: one row per brief for the UI. | `BriefingEntity` events | `BriefingEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record MeetingRequest(
    String briefId,
    String counterpartyName,
    Instant meetingAt,
    List<String> attendees,
    List<String> agendaTopics,
    String requestedBy,
    Instant requestedAt,
    CrmSnapshot crmSnapshot       // raw; PII-sanitizer input
) {}

record CrmSnapshot(
    String contactName,           // raw PII
    String contactEmail,          // raw PII
    String contactPhone,          // raw PII
    String dealStage,
    String lastInteractionNote,
    Instant lastContactAt
) {}

record SanitizedCrmData(
    String redactedContactName,
    String redactedContactEmail,
    String redactedContactPhone,
    String dealStage,
    String lastInteractionNote,   // already scrubbed of identifiers
    Instant lastContactAt,
    List<String> piiCategoriesFound
) {}

record TalkingPoint(
    String topic,
    String point,
    String evidenceSource          // e.g. "CRM note 2026-05-12", "SEC 10-Q Q1 2026"
) {}

record RiskFlag(
    String label,
    RiskLevel level,
    String detail
) {}
enum RiskLevel { LOW, MEDIUM, HIGH }

record MeetingBrief(
    String executiveSummary,
    List<TalkingPoint> talkingPoints,
    List<RiskFlag> riskFlags,
    String financialHighlights,
    List<String> recentNewsItems,
    Instant preparedAt
) {}

record BriefEval(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record Briefing(
    String briefId,
    Optional<MeetingRequest> request,
    Optional<SanitizedCrmData> sanitized,
    Optional<MeetingBrief> brief,
    Optional<BriefEval> eval,
    BriefingStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum BriefingStatus {
    REQUESTED, DATA_SANITIZED, BRIEFING, BRIEF_READY, EVALUATED, FAILED
}
```

Events on `BriefingEntity`: `MeetingRequested`, `ContactDataSanitized`, `BriefingStarted`, `BriefReady`, `BriefEvaluated`, `BriefingFailed`.

Every nullable lifecycle field on the `Briefing` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/briefs` — body `{ counterpartyName, meetingAt, attendees: [String], agendaTopics: [String], requestedBy, crmSnapshot: CrmSnapshot }` → `{ briefId }`.
- `GET /api/briefs` — list all briefs, newest-first.
- `GET /api/briefs/{id}` — one brief.
- `GET /api/briefs/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Meeting Preparer</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted briefs (status pill + eval score chip + counterparty name + age) and a right pane with the selected brief's detail — meeting metadata, sanitized CRM preview, executive summary, talking-points list, risk flags, financial highlights, and recent news.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `ContactSanitizer` Consumer): redacts contact names, email addresses, phone numbers, and account-like identifiers from the raw CRM snapshot before any LLM call. Records which categories were found.

The evaluator (`BriefingEvaluator`) is a deterministic rule-based scorer that runs in `evalStep` — not an LLM call. It checks that every talking point has a non-empty `evidenceSource`, that at least one risk flag is present when the financial highlights mention losses or litigation, and that `recentNewsItems` is not empty. Emits `BriefEvaluated` with a 1–5 score and a one-line rationale.

## 9. Agent prompts

- `BriefingAgent` → `prompts/briefing-agent.md`. The single decision-making LLM. System prompt instructs it to read the sanitized CRM snapshot attachment, the financial highlights attachment, and the news items attachment, then produce a `MeetingBrief` for the given meeting context.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a meeting request for a seeded counterparty; within 30 s the brief appears with all sections populated and an eval score chip.
2. **J2** — A CRM snapshot containing `alice.wong@example.com` and `+1-415-555-0102` is submitted; the LLM call log shows only `[REDACTED-EMAIL]` and `[REDACTED-PHONE]`; the entity's `request.crmSnapshot` retains the raw values for audit.
3. **J3** — A brief whose talking points all carry empty `evidenceSource` strings receives an eval score of 1 with a clear rationale; the UI flags the card with a red border.
4. **J4** — Three meeting requests submitted in succession each produce independent briefs; the live list shows all three cards updating concurrently via SSE.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named meeting-preparer demonstrating the single-agent × finance-analysis cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-finance-analysis-meeting-preparer. Java package io.akka.samples.meetingpreparer.
Akka 3.6.0. HTTP port 9818.

Components to wire (exactly):

- 1 AutonomousAgent BriefingAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/briefing-agent.md>) and
  .capability(TaskAcceptance.of(PREPARE_BRIEF).maxIterationsPerTask(3)). The task receives
  meeting metadata in its instruction text and three attachments: sanitized-crm.txt (the
  redacted CRM snapshot), financial-highlights.txt (seeded financial data), and
  news-items.txt (seeded news lines). Output: MeetingBrief{executiveSummary: String,
  talkingPoints: List<TalkingPoint>, riskFlags: List<RiskFlag>, financialHighlights: String,
  recentNewsItems: List<String>, preparedAt: Instant}.

- 1 Workflow BriefingWorkflow per briefId with three steps:
  * awaitSanitizedStep — polls BriefingEntity.getBriefing every 1s; on
    briefing.sanitized().isPresent() advances to briefStep.
    WorkflowSettings.stepTimeout 15s.
  * briefStep — emits BriefingStarted, then calls componentClient.forAutonomousAgent(
    BriefingAgent.class, "briefer-" + briefId).runSingleTask(
      TaskDef.instructions(formatMeetingContext(briefing))
        .attachment("sanitized-crm.txt", formatSanitizedCrm(briefing.sanitized()).getBytes())
        .attachment("financial-highlights.txt", lookupFinancialHighlights(
            briefing.request().counterpartyName()).getBytes())
        .attachment("news-items.txt", lookupNewsItems(
            briefing.request().counterpartyName()).getBytes())
    ) — returns a taskId, then forTask(taskId).result(PREPARE_BRIEF) to fetch the brief.
    On success calls BriefingEntity.recordBrief(brief). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(2).failoverTo(BriefingWorkflow::error).
  * evalStep — runs a deterministic rule-based BriefingEvaluator (NOT an LLM call) over
    the recorded brief: checks that every TalkingPoint.evidenceSource is non-empty, that
    recentNewsItems is not empty, and that at least one RiskFlag is present. Emits
    BriefEvaluated{score: 1-5, rationale: String}. WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity BriefingEntity (one per briefId). State Briefing{briefId: String,
  request: Optional<MeetingRequest>, sanitized: Optional<SanitizedCrmData>,
  brief: Optional<MeetingBrief>, eval: Optional<BriefEval>, status: BriefingStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. BriefingStatus enum: REQUESTED,
  DATA_SANITIZED, BRIEFING, BRIEF_READY, EVALUATED, FAILED. Events:
  MeetingRequested{request}, ContactDataSanitized{sanitized}, BriefingStarted{},
  BriefReady{brief}, BriefEvaluated{eval}, BriefingFailed{reason}.
  Commands: request, attachSanitizedData, markBriefing, recordBrief, recordEval, fail,
  getBriefing. emptyState() returns Briefing.initial("") with no commandContext()
  reference (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state
  and Optional.of(...) inside the event-applier.

- 1 Consumer ContactSanitizer subscribed to BriefingEntity events; on MeetingRequested
  runs a regex+heuristic redaction pipeline (emails, phone numbers, person names,
  account-id-like tokens) over crmSnapshot.contactName, crmSnapshot.contactEmail,
  crmSnapshot.contactPhone, and crmSnapshot.lastInteractionNote, computes the list of
  categories found, builds SanitizedCrmData, then calls
  BriefingEntity.attachSanitizedData(sanitized). After attachSanitizedData lands, the
  same Consumer starts a BriefingWorkflow with id = "briefing-" + briefId.

- 1 View BriefingView with row type BriefingRow (mirrors Briefing minus
  request.crmSnapshot.contactEmail, contactPhone, contactName — the audit log keeps the
  raw; the view holds the sanitized form for the UI). Table updater consumes
  BriefingEntity events. ONE query getAllBriefings: SELECT * AS briefings FROM
  briefing_view. No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2);
  caller filters client-side.

- 2 HttpEndpoints:
  * BriefingEndpoint at /api with POST /briefs (body
    {counterpartyName, meetingAt, attendees: [String], agendaTopics: [String],
    requestedBy, crmSnapshot: {contactName, contactEmail, contactPhone, dealStage,
    lastInteractionNote, lastContactAt}}; mints briefId; calls BriefingEntity.request;
    returns {briefId}), GET /briefs (list from getAllBriefings, sorted newest-first),
    GET /briefs/{id} (one row), GET /briefs/sse (Server-Sent Events forwarded from the
    view's stream-updates), and three /api/metadata/* endpoints serving the YAML/MD
    files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- BriefingTasks.java declaring one Task<R> constant: PREPARE_BRIEF = Task.name("Prepare
  meeting brief").description("Read the sanitized CRM, financial, and news attachments and
  produce a MeetingBrief for the upcoming meeting").resultConformsTo(MeetingBrief.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records MeetingRequest, CrmSnapshot, SanitizedCrmData, TalkingPoint, RiskFlag,
  RiskLevel, MeetingBrief, BriefEval, Briefing, BriefingStatus.

- BriefingEvaluator.java — pure deterministic logic (no LLM). Inputs: MeetingBrief.
  Outputs: BriefEval. Scoring rubric: +1 if every talkingPoint.evidenceSource non-empty;
  +1 if recentNewsItems non-empty; +1 if riskFlags non-empty; +1 if
  talkingPoints.size() >= 3; +1 if executiveSummary length >= 80 chars. Score 1–5.
  Documented in Javadoc on the class.

- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9818 and the three model-provider blocks (anthropic
  claude-sonnet-4-6, openai gpt-4o, googleai-gemini gemini-2.5-flash) reading the
  canonical env vars ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. The BriefingAgent.definition() binds the configured
  provider via .modelProvider("${akka.javasdk.agent.default}") or the per-agent override
  pattern from the akka-context docs.

- src/main/resources/sample-events/counterparties.jsonl with 3 seeded counterparty
  profiles: a private-equity-backed mid-market tech company, a regional insurance carrier,
  and a manufacturing holdco. Each carries a CrmSnapshot with realistic PII strings
  (contact name, email, phone) and a lastInteractionNote, plus financial-highlights and
  news-items blocks so the agent has grounded content.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (S1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = recommend-only
  (the agent's brief is advisory), oversight.human_in_loop = true (the banker reads the
  brief before the call), failure.failure_modes including "hallucinated-financial-data",
  "missed-risk-flag", "pii-leakage-via-llm", "stale-data-used-as-current";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/briefing-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Meeting Preparer", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of brief cards; right = selected-brief detail with meeting metadata,
  sanitized CRM preview, executive summary, talking points, risk flags, financial
  highlights, recent news, and eval-score chip). Browser title exactly:
  <title>Akka Sample: Meeting Preparer</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        shape-correct outputs per agent.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json, picks one entry pseudo-randomly per
  call (seedFor(briefId)), and deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    prepare-meeting-brief.json — 6 MeetingBrief entries covering all three seeded
    counterparties. Each entry has a 2-sentence executiveSummary, 3 TalkingPoints each
    with a non-empty evidenceSource, 1-2 RiskFlags, a financialHighlights line, and
    2-3 recentNewsItems. Plus 1 deliberately incomplete entry with empty
    evidenceSource strings on all TalkingPoints — used to exercise eval score 1 (J3).
  A MockModelProvider.seedFor(briefId) helper makes selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. BriefingAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. BriefingTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (briefStep 60s,
  awaitSanitizedStep 15s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Briefing record is Optional<T>.
- Lesson 7: BriefingTasks.java with PREPARE_BRIEF mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9818 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND the
  mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements in the DOM.
- The single-agent invariant: exactly ONE AutonomousAgent (BriefingAgent). The
  on-decision eval is rule-based (BriefingEvaluator.java) and does NOT make an LLM call.
- CRM data is passed as task ATTACHMENTS, never inlined into the agent's instruction text.
  Verify the generated briefStep uses TaskDef.attachment(...) for sanitized-crm.txt,
  financial-highlights.txt, and news-items.txt.
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

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
