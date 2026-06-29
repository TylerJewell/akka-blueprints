# SPEC — wellness-check-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Wellness Check Agent.
**One-line pitch:** An HR coordinator schedules a morale check-in campaign; one AI agent reads each employee's response (passed as a task attachment, never as inline prompt text) and returns a structured `CheckInAnalysis` — morale level, sentiment, crisis flag, and one actionable recommendation.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the hr-recruiting domain. One `WellnessCheckAgent` (AutonomousAgent) carries the entire analysis; the surrounding components only prepare its input, enforce safety on its output, and aggregate results for human review. Three governance mechanisms are wired around the agent:

- A **special-category sanitizer** runs inside a Consumer between the raw employee response and the agent call — so the model never sees identifiable health, mental-health, or disability markers.
- A **before-agent-response guardrail** validates the agent's analysis on every turn: well-formed JSON, morale level is within the allowed enum, crisis flag is a boolean, and the recommendation is non-empty. A malformed analysis triggers a retry inside the same task. Additionally, if the candidate response contains a `crisisFlag: true`, the guardrail intercepts and emits a `CrisisEscalated` event before letting the workflow continue to the human-on-the-loop path.
- A **post-market-surveillance human-on-the-loop** review aggregates morale signal distributions at the campaign level — flagging campaigns where the proportion of LOW or CRISIS-tagged responses crosses a threshold for human workforce-risk review.

The blueprint demonstrates that a single-agent pattern does not mean ungoverned: three independent checks sit on either side of the one decision-making LLM call, and the most sensitive path (crisis detection) is always escalated to a human before any result is acted upon.

## 3. User-facing flows

The user opens the App UI tab.

1. The HR coordinator fills in a **Campaign** form: campaign name, target audience label, a set of check-in questions (up to 5), and a scheduled-send timestamp.
2. The coordinator clicks **Launch campaign**. The UI POSTs to `/api/campaigns` and receives a `campaignId`.
3. The campaign card appears in `SCHEDULED` state. When the send time arrives (or immediately if no future time was set), the campaign transitions to `ACTIVE` and the system emits a simulated employee response per configured participant.
4. Each simulated response enters as a `POST /api/check-ins`. A new check-in card appears in `RECEIVED` state. Within ~1 s it transitions to `SANITIZED` — the redacted text is visible, with a list of special-category markers the sanitizer found.
5. Within ~10–30 s the workflow's `analyseStep` completes. The card transitions to `ANALYSING` then `ANALYSIS_RECORDED`. The analysis appears: a morale-level badge (`HIGH` / `MODERATE` / `LOW` / `CRISIS`), a short interpretation paragraph, and a recommendation.
6. If `crisisFlag` is `true`, the card immediately transitions to `ESCALATED` — it is removed from the normal morale list and appears in a dedicated **Crisis queue** with a banner: "Routed to human. Do not act without review."
7. The coordinator can view the campaign's aggregate morale distribution chart (HIGH / MODERATE / LOW counts). If the LOW + CRISIS proportion exceeds the configured threshold, the campaign card shows a **Risk review required** banner.
8. The coordinator submits another campaign; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `CheckInEndpoint` | `HttpEndpoint` | `/api/campaigns/*`, `/api/check-ins/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `CampaignEntity`, `CheckInEntity`, `CheckInView`, `CampaignView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `CampaignEntity` | `EventSourcedEntity` | Per-campaign lifecycle: scheduled → active → completed. | `CheckInEndpoint` | `CampaignView`, `CheckInEntity` |
| `CheckInEntity` | `EventSourcedEntity` | Per-check-in lifecycle: received → sanitized → analysing → analysis recorded → escalated or evaluated. Source of truth. | `CheckInEndpoint`, `ResponseSanitizer`, `CheckInWorkflow` | `CheckInView` |
| `ResponseSanitizer` | `Consumer` | Subscribes to `ResponseReceived` events; redacts special-category markers; calls `CheckInEntity.attachSanitized`. | `CheckInEntity` events | `CheckInEntity`, `CheckInWorkflow` |
| `CheckInWorkflow` | `Workflow` | One workflow per check-in. Steps: `awaitSanitizedStep` → `analyseStep` → `surveillanceStep`. | started by `ResponseSanitizer` once sanitized event lands | `WellnessCheckAgent`, `CheckInEntity` |
| `WellnessCheckAgent` | `AutonomousAgent` | The one decision-making LLM. Receives check-in questions in the task definition and the sanitized response as a task attachment; returns `CheckInAnalysis`. | invoked by `CheckInWorkflow` | returns analysis |
| `ResponseGuardrail` | supporting class | `before-agent-response` hook on `WellnessCheckAgent`. Validates structure, enum values, and intercepts crisis. | `WellnessCheckAgent` | `WellnessCheckAgent` loop |
| `MoraleSurveillance` | supporting class | Deterministic rule evaluator. Computes campaign-level risk signal from aggregated morale counts. | invoked by `CheckInWorkflow.surveillanceStep` | `CheckInEntity` |
| `CheckInView` | `View` | Read model: one row per check-in for the UI. | `CheckInEntity` events | `CheckInEndpoint` |
| `CampaignView` | `View` | Read model: one row per campaign with aggregate morale counts. | `CampaignEntity` events, `CheckInEntity` events | `CheckInEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record CheckInQuestion(String questionId, String text, QuestionKind kind) {}
enum QuestionKind { OPEN_ENDED, LIKERT_1_5, YES_NO }

record CampaignRequest(
    String campaignId,
    String campaignName,
    String audienceLabel,
    List<CheckInQuestion> questions,
    Instant scheduledAt,
    String createdBy
) {}

record EmployeeResponse(
    String checkInId,
    String campaignId,
    String employeeRef,       // pseudonymised reference, not a name
    Map<String, String> answers,  // questionId -> raw answer text
    Instant receivedAt
) {}

record SanitizedResponse(
    Map<String, String> redactedAnswers,  // same keys, redacted values
    List<String> specialCategoriesFound  // e.g. ["mental-health-marker","disability-marker"]
) {}

record CheckInAnalysis(
    MoraleLevel moraleLevel,
    String interpretation,    // 1–3 sentences
    boolean crisisFlag,
    String recommendation,
    Instant analysedAt
) {}
enum MoraleLevel { HIGH, MODERATE, LOW, CRISIS }

record SurveillanceResult(
    boolean riskFlagRaised,
    String rationale,
    Instant evaluatedAt
) {}

record CheckIn(
    String checkInId,
    String campaignId,
    Optional<EmployeeResponse> response,
    Optional<SanitizedResponse> sanitized,
    Optional<CheckInAnalysis> analysis,
    Optional<SurveillanceResult> surveillance,
    CheckInStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum CheckInStatus {
    RECEIVED, SANITIZED, ANALYSING, ANALYSIS_RECORDED, ESCALATED, EVALUATED, FAILED
}

record Campaign(
    String campaignId,
    Optional<CampaignRequest> request,
    CampaignStatus status,
    int responsesReceived,
    int highCount,
    int moderateCount,
    int lowCount,
    int crisisCount,
    Instant createdAt,
    Optional<Instant> completedAt
) {}

enum CampaignStatus { SCHEDULED, ACTIVE, COMPLETED, FAILED }
```

Events on `CheckInEntity`: `ResponseReceived`, `ResponseSanitized`, `AnalysisStarted`, `AnalysisRecorded`, `CrisisEscalated`, `SurveillanceEvaluated`, `CheckInFailed`.

Events on `CampaignEntity`: `CampaignScheduled`, `CampaignActivated`, `CampaignCompleted`, `CampaignFailed`.

Every nullable lifecycle field on `CheckIn` and `Campaign` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/campaigns` — body `{ campaignName, audienceLabel, questions: [CheckInQuestion], scheduledAt, createdBy }` → `{ campaignId }`.
- `GET /api/campaigns` — list all campaigns, newest-first.
- `GET /api/campaigns/{id}` — one campaign with aggregate morale counts.
- `POST /api/check-ins` — body `{ campaignId, employeeRef, answers: { questionId: answerText } }` → `{ checkInId }`.
- `GET /api/check-ins` — list all check-ins, newest-first.
- `GET /api/check-ins/{id}` — one check-in.
- `GET /api/check-ins/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Wellness Check Agent</title>`.

The App UI tab is a two-column layout: a left rail with the campaign list and the live check-in list (status pill + morale badge + age), and a right pane with the selected check-in's detail — questions sent, sanitized response text, analysis summary, morale badge, recommendation, and crisis banner if escalated.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — Special-category sanitizer** (`special-category`, applied inside `ResponseSanitizer` Consumer): redacts mental-health markers, disability-related language, physical health indicators, and emotional-state identifiers from the raw employee response text before any LLM call. Records which categories were found. The raw response is preserved on the entity for audit.
- **G1 — before-agent-response guardrail**: runs on every turn of `WellnessCheckAgent`. Asserts the candidate response is well-formed `CheckInAnalysis` JSON, `moraleLevel` is in `{HIGH, MODERATE, LOW, CRISIS}`, `crisisFlag` is a boolean, and `recommendation` is non-empty. If `crisisFlag == true`, the guardrail emits `CrisisEscalated` on `CheckInEntity` before accepting the response, ensuring crisis routing happens at the hook boundary before the response is returned to the workflow. On structural failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.
- **H1 — post-market-surveillance human-on-the-loop**: runs as `surveillanceStep` inside the workflow after `AnalysisRecorded` lands. `MoraleSurveillance` is deterministic and rule-based (no LLM call). It reads the parent `CampaignView` row and checks whether the campaign's (lowCount + crisisCount) / responsesReceived ratio exceeds 0.25. If so, it raises a risk flag on the campaign and emits `SurveillanceEvaluated{riskFlagRaised: true}` with a rationale naming the ratio. The campaign card shows a **Risk review required** banner. A human HR decision-maker reviews the aggregate before acting.

## 9. Agent prompts

- `WellnessCheckAgent` → `prompts/wellness-check-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached sanitized response, consider the check-in questions that were asked, and return a `CheckInAnalysis` with a morale level, an interpretation, a crisis flag, and an actionable recommendation.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Coordinator launches a campaign with the seeded 5-question pulse; within 30 s each simulated check-in reaches `EVALUATED` with a morale badge and a non-empty recommendation.
2. **J2** — The agent's first response on a check-in carries an out-of-enum `moraleLevel` (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration produces a valid analysis; the UI never displays the malformed response.
3. **J3** — A check-in response containing explicit crisis language produces `crisisFlag: true`; the check-in transitions to `ESCALATED`; the check-in card disappears from the standard morale list and appears only in the crisis queue.
4. **J4** — Raw response text containing `[mental-health-marker]` and `[disability-reference]` test tokens is submitted; the LLM call log shows only the redacted forms; `response.answers` on the entity retains the raw text.
5. **J5** — A campaign whose simulated responses produce a LOW + CRISIS ratio above 25 % receives a `riskFlagRaised: true` `SurveillanceResult`; the campaign card shows the **Risk review required** banner.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named wellness-check-agent demonstrating the single-agent × hr-recruiting cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-hr-recruiting-employee-wellness-check. Java package
io.akka.samples.wellnesscheckagent. Akka 3.6.0. HTTP port 9338.

Components to wire (exactly):

- 1 AutonomousAgent WellnessCheckAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/wellness-check-agent.md>) and
  .capability(TaskAcceptance.of(ANALYSE_CHECK_IN).maxIterationsPerTask(3)). The task receives
  the check-in questions as its instruction text and the sanitized response as a task ATTACHMENT
  (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the canonical
  call). Output: CheckInAnalysis{moraleLevel: MoraleLevel (HIGH/MODERATE/LOW/CRISIS),
  interpretation: String, crisisFlag: boolean, recommendation: String, analysedAt: Instant}.
  The agent is configured with a before-agent-response guardrail (see G1 in eval-matrix.yaml)
  registered via the agent's guardrail-configuration block. On guardrail rejection the agent
  loop retries the response within its 3-iteration budget.

- 1 Workflow CheckInWorkflow per checkInId with three steps:
  * awaitSanitizedStep — polls CheckInEntity.getCheckIn every 1 s; on checkIn.sanitized()
    .isPresent() advances to analyseStep. WorkflowSettings.stepTimeout 15 s (sanitizer is
    in-process and fast).
  * analyseStep — emits AnalysisStarted, then calls componentClient.forAutonomousAgent(
    WellnessCheckAgent.class, "wellness-" + checkInId).runSingleTask(
      TaskDef.instructions(formatQuestions(campaign.questions))
        .attachment("response.txt", sanitized.redactedAnswers serialised as plain text)
    ) — returns a taskId, then forTask(taskId).result(ANALYSE_CHECK_IN) to fetch the analysis.
    On success calls CheckInEntity.recordAnalysis(analysis). WorkflowSettings.stepTimeout 60 s
    with defaultStepRecovery maxRetries(2).failoverTo(CheckInWorkflow::error).
  * surveillanceStep — runs MoraleSurveillance (NOT an LLM call) over the campaign's
    aggregate morale counts. Checks whether (lowCount + crisisCount) / responsesReceived > 0.25.
    Emits SurveillanceEvaluated{riskFlagRaised, rationale, evaluatedAt}. WorkflowSettings
    .stepTimeout 5 s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity CheckInEntity (one per checkInId). State CheckIn{checkInId: String,
  campaignId: String, response: Optional<EmployeeResponse>, sanitized: Optional<SanitizedResponse>,
  analysis: Optional<CheckInAnalysis>, surveillance: Optional<SurveillanceResult>,
  status: CheckInStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  CheckInStatus enum: RECEIVED, SANITIZED, ANALYSING, ANALYSIS_RECORDED, ESCALATED, EVALUATED,
  FAILED. Events: ResponseReceived{response}, ResponseSanitized{sanitized}, AnalysisStarted{},
  AnalysisRecorded{analysis}, CrisisEscalated{reason}, SurveillanceEvaluated{surveillance},
  CheckInFailed{reason}. Commands: receive, attachSanitized, markAnalysing, recordAnalysis,
  escalateCrisis, recordSurveillance, fail, getCheckIn. emptyState() returns
  CheckIn.initial("", "") with no commandContext() reference (Lesson 3). Every Optional<T>
  field uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 EventSourcedEntity CampaignEntity (one per campaignId). State Campaign{campaignId: String,
  request: Optional<CampaignRequest>, status: CampaignStatus, responsesReceived: int,
  highCount: int, moderateCount: int, lowCount: int, crisisCount: int, createdAt: Instant,
  completedAt: Optional<Instant>}. Events: CampaignScheduled{request}, CampaignActivated{},
  CampaignCompleted{}, CampaignFailed{reason}. Commands: schedule, activate, complete, fail,
  getCampaign. emptyState() returns Campaign.initial("") with no commandContext() reference.

- 1 Consumer ResponseSanitizer subscribed to CheckInEntity events; on ResponseReceived runs
  a regex+heuristic redaction pipeline over each answer value in employeeResponse.answers.
  Redaction categories: mental-health-marker (depression, anxiety, burnout, panic, self-harm
  related terms), disability-marker, physical-health-indicator, emotional-state-identifier.
  Builds SanitizedResponse with redactedAnswers and specialCategoriesFound. Calls
  CheckInEntity.attachSanitized(sanitized). After attachSanitized lands, same Consumer
  starts a CheckInWorkflow with id = "checkin-" + checkInId.

- 1 View CheckInView with row type CheckInRow (mirrors CheckIn minus
  response.answers raw values — the audit log keeps raw; the view holds the sanitized form).
  Table updater consumes CheckInEntity events. ONE query getAllCheckIns:
  SELECT * AS checkIns FROM check_in_view. No WHERE status filter — Akka cannot auto-index
  enum columns (Lesson 2); caller filters client-side.

- 1 View CampaignView with row type CampaignRow (mirrors Campaign). Table updater consumes
  CampaignEntity events and CheckInEntity AnalysisRecorded events (to increment morale counts).
  ONE query getAllCampaigns: SELECT * AS campaigns FROM campaign_view.

- 2 HttpEndpoints:
  * CheckInEndpoint at /api with:
    POST /campaigns (body {campaignName, audienceLabel, questions: [{questionId, text, kind}],
    scheduledAt, createdBy}; mints campaignId; calls CampaignEntity.schedule; returns {campaignId})
    GET /campaigns (list from getAllCampaigns, sorted newest-first)
    GET /campaigns/{id} (one campaign row)
    POST /check-ins (body {campaignId, employeeRef, answers: {questionId: answerText}};
    mints checkInId; calls CheckInEntity.receive; returns {checkInId})
    GET /check-ins (list all check-ins, newest-first)
    GET /check-ins/{id} (one check-in row)
    GET /check-ins/sse (Server-Sent Events forwarded from CheckInView stream-updates)
    GET /api/metadata/* (serving the YAML/MD files from src/main/resources/metadata/)
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- CheckInTasks.java declaring one Task<R> constant: ANALYSE_CHECK_IN = Task.name("Analyse
  check-in response").description("Read the attached sanitized employee response and produce
  a CheckInAnalysis with morale level, interpretation, crisis flag, and recommendation")
  .resultConformsTo(CheckInAnalysis.class). DO NOT skip this — the AutonomousAgent requires
  its companion Tasks class (Lesson 7).

- Domain records CheckInQuestion, QuestionKind, CampaignRequest, EmployeeResponse,
  SanitizedResponse, CheckInAnalysis, MoraleLevel, SurveillanceResult, CheckIn, CheckInStatus,
  Campaign, CampaignStatus.

- ResponseGuardrail.java implementing the before-agent-response hook. Reads the candidate
  CheckInAnalysis from the LLM response, runs the three checks listed in eval-matrix.yaml G1
  (moraleLevel enum, crisisFlag boolean, recommendation non-empty), and for crisisFlag == true
  emits CrisisEscalated on CheckInEntity before returning Guardrail.accept(). On structural
  failure returns Guardrail.reject(<structured-error>) to force the agent loop to retry.

- MoraleSurveillance.java — pure deterministic logic (no LLM). Inputs: CampaignRow (aggregate
  counts). Outputs: SurveillanceResult. Ratio threshold: 0.25. Documented in Javadoc on class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9338 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/questions.jsonl with 3 seeded question sets:
  a 5-question morale pulse, a 4-question burnout assessment, and a 3-question return-to-office
  sentiment survey.

- src/main/resources/sample-events/seed-responses.jsonl with 3 paired example employee
  responses. Each contains 2–3 special-category marker terms so S1 has work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (S1, G1, H1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. Regulation anchor GDPR Art. 9
  referenced for S1 and H1.

- risk-survey.yaml at the project root with data.data_classes.special_category_wellness = true,
  special_category_handled_by_sanitizer_before_llm = true,
  decisions.authority_level = recommend-only (the agent's analysis is advisory, not enforced),
  oversight.human_on_loop = true (aggregate surveillance reviewed by human HR decision-maker),
  failure.failure_modes including "crisis-missed", "morale-miscalibration",
  "special-category-leakage-via-llm", "false-crisis-escalation"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/wellness-check-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Wellness Check Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs. App UI tab uses a two-column layout (left = campaign list + live check-in
  cards; right = selected check-in detail with questions sent, sanitized response text,
  analysis summary, morale badge, recommendation, crisis banner if escalated).
  Browser title exactly: <title>Akka Sample: Wellness Check Agent</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing ModelProvider. Per-task dispatch on the Task<R>
  id. Each branch reads src/main/resources/mock-responses/<task-id>.json, picks one entry
  pseudo-randomly per call (seedFor(checkInId)), deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    analyse-check-in.json — 8 CheckInAnalysis entries covering all four MoraleLevel values.
      Each entry has a non-empty interpretation paragraph and an actionable recommendation.
      MoraleLevel values vary (2 HIGH, 3 MODERATE, 2 LOW, 1 CRISIS). crisisFlag is true only
      on the CRISIS entry. Plus 2 deliberately MALFORMED entries (one with a moraleLevel value
      outside the enum; one with a null recommendation) — the guardrail blocks both, exercising
      the retry path. The mock selects a malformed entry on the FIRST iteration of every 3rd
      check-in (modulo seed) so J2 is reproducible.
- MockModelProvider.seedFor(checkInId) helper makes per-check-in selection deterministic.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded. WellnessCheckAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion CheckInTasks.java MUST exist.
- Lesson 4: every workflow step has explicit stepTimeout (analyseStep 60 s, awaitSanitizedStep
  15 s, surveillanceStep 5 s, error 5 s).
- Lesson 6: every nullable lifecycle field on CheckIn and Campaign records is Optional<T>.
- Lesson 7: CheckInTasks.java with ANALYSE_CHECK_IN is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash.
- Lesson 9: run command is "/akka:build".
- Lesson 10: port 9338 declared explicitly in application.conf.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words.
- Lesson 24: generated static-resources/index.html includes mermaid CSS overrides AND
  themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching by data-tab / data-panel attribute. Exactly five tab-panel sections.
- Single-agent invariant: exactly ONE AutonomousAgent (WellnessCheckAgent). MoraleSurveillance
  is rule-based (no LLM call).
- The sanitized response is passed as a Task ATTACHMENT, never inlined into instructions.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as a post-return check.
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
