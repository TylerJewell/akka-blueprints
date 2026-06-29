# SPEC — personal-assistant

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** PersonalAssistant.
**One-line pitch:** A user sends a natural-language request (e.g. "add a reminder to pack sunscreen before my Friday flight"); one AI agent reads the user's current calendar and task context (passed as a task attachment, never as inline prompt text) and returns a structured `AssistantAction` — a typed description of what to create, update, or mark complete.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the planning-travel domain. One `AssistantAgent` (AutonomousAgent) carries the entire decision; the surrounding components prepare its input and validate its output. Two governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw context submission and the agent call — so the model never sees raw contact names, email addresses, phone numbers, or account identifiers present in calendar metadata.
- A **before-tool-call guardrail** intercepts every proposed calendar or task write before execution: checks that the target date is in the future, the action type is in the allowed set, and the title is non-empty. An invalid write is rejected with a structured error; the agent retries within its iteration budget.

The blueprint shows that the single-agent pattern does not mean "ungoverned" — two independent checks sit on either side of the one decision-making LLM call.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a request into the **Request** textarea (e.g. "Schedule a hotel check-in reminder for next Monday at 9 AM" or "Mark the airport transfer task as complete").
2. The user picks a **context profile** from a dropdown (Travel week, Day-trip, Minimal) or pastes a custom JSON context snapshot (calendar events + task list).
3. The user clicks **Send request**. The UI POSTs to `/api/requests` and receives a `requestId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s it transitions to `SANITIZED` — the redacted context is visible in the card detail, with a small list of PII categories the sanitizer found.
5. Within ~10–30 s the workflow's `actStep` completes. The card transitions to `ACTING` then `ACTION_RECORDED`. The action appears: a top-level action type badge (`CREATE_EVENT` / `CREATE_TASK` / `UPDATE_TASK` / `COMPLETE_TASK` / `SET_REMINDER` / `NO_ACTION`), a short explanation paragraph, and a change-set table (field, old value, new value).
6. Within ~1 s of the action, the `evalStep` finishes. The card shows an **eval score** chip (1–5) plus a one-line rationale describing whether the action is grounded and specific.
7. The user can send another request; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AssistantEndpoint` | `HttpEndpoint` | `/api/requests/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `AssistantEntity`, `AssistantView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `AssistantEntity` | `EventSourcedEntity` | Per-request lifecycle: submitted → sanitized → acting → action recorded → evaluated. Source of truth. | `AssistantEndpoint`, `ContextSanitizer`, `AssistantWorkflow` | `AssistantView` |
| `ContextSanitizer` | `Consumer` | Subscribes to `RequestSubmitted` events; redacts PII from context snapshot; calls `AssistantEntity.attachSanitized`. | `AssistantEntity` events | `AssistantEntity` |
| `AssistantWorkflow` | `Workflow` | One workflow per request. Steps: `awaitSanitizedStep` → `actStep` → `evalStep`. | started by `ContextSanitizer` once sanitized event lands | `AssistantAgent`, `AssistantEntity` |
| `AssistantAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the natural-language request as task instructions and the sanitized context snapshot as a task attachment; returns `AssistantAction`. | invoked by `AssistantWorkflow` | returns action |
| `WriteGuardrail` | supporting class | `before-tool-call` hook registered on `AssistantAgent`. Validates every proposed write before execution. | `AssistantAgent` | `AssistantAgent` |
| `ActionScorer` | supporting class | Deterministic rule-based scorer. Runs in `evalStep`. No LLM call. | `AssistantWorkflow` | `AssistantWorkflow` |
| `AssistantView` | `View` | Read model: one row per request for the UI. | `AssistantEntity` events | `AssistantEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ContextEvent(
    String eventId,
    String title,
    LocalDate date,
    Optional<LocalTime> time,
    String location
) {}

record ContextTask(
    String taskId,
    String title,
    boolean completed,
    Optional<LocalDate> dueDate,
    Priority priority
) {}
enum Priority { LOW, NORMAL, HIGH }

record ContextSnapshot(
    List<ContextEvent> events,
    List<ContextTask> tasks,
    String timezone
) {}

record SanitizedContext(
    String redactedSnapshot,   // JSON with PII redacted
    List<String> piiCategoriesFound
) {}

record ChangeField(
    String field,
    Optional<String> oldValue,
    String newValue
) {}

record AssistantAction(
    ActionType actionType,
    String explanation,
    List<ChangeField> changeSet,
    Instant decidedAt
) {}
enum ActionType {
    CREATE_EVENT, CREATE_TASK, UPDATE_TASK, COMPLETE_TASK, SET_REMINDER, NO_ACTION
}

record EvalResult(
    int score,          // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record AssistantRequest(
    String requestId,
    String naturalLanguageRequest,
    ContextSnapshot rawContext,
    String submittedBy,
    Instant submittedAt
) {}

record AssistantState(
    String requestId,
    Optional<AssistantRequest> request,
    Optional<SanitizedContext> sanitized,
    Optional<AssistantAction> action,
    Optional<EvalResult> eval,
    RequestStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RequestStatus {
    SUBMITTED, SANITIZED, ACTING, ACTION_RECORDED, EVALUATED, FAILED
}
```

Events on `AssistantEntity`: `RequestSubmitted`, `ContextSanitized`, `ActionStarted`, `ActionRecorded`, `EvaluationScored`, `RequestFailed`.

Every nullable lifecycle field on the `AssistantState` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/requests` — body `{ naturalLanguageRequest, rawContext: ContextSnapshot, submittedBy }` → `{ requestId }`.
- `GET /api/requests` — list all requests, newest-first.
- `GET /api/requests/{id}` — one request.
- `GET /api/requests/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: PersonalAssistant</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted requests (status pill + action type badge + age) and a right pane with the selected request's detail — original natural-language request, sanitized context preview, action type badge, explanation paragraph, change-set table, and eval-score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs on every proposed write call inside `AssistantAgent`. Asserts (1) the `actionType` is in `{CREATE_EVENT, CREATE_TASK, UPDATE_TASK, COMPLETE_TASK, SET_REMINDER, NO_ACTION}`, (2) any `date` field on a new event is not in the past, (3) any `title` or `taskTitle` field is non-empty, and (4) the write is scoped to the authenticated user's context (no cross-user writes). On failure, returns a structured `invalid-tool-call` error to the agent loop so the task retries within its iteration budget.
- **S1 — PII sanitizer** (`pii`, applied inside `ContextSanitizer` Consumer): redacts contact email addresses, phone numbers, physical addresses, person names appearing in event attendee lists or task descriptions, and account-like identifiers from the raw context snapshot before any LLM call. Records which categories were found.

## 9. Agent prompts

- `AssistantAgent` → `prompts/assistant-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached context snapshot, interpret the natural-language request, and return one `AssistantAction` describing the minimal change needed to satisfy the request.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits "Schedule a hotel check-in reminder for Monday at 9 AM" with the Travel week context; within 30 s the action appears as `SET_REMINDER` with a non-empty change set and an eval score chip.
2. **J2** — The agent proposes a `CREATE_EVENT` with a date in the past (mock LLM path); the `before-tool-call` guardrail rejects it; the agent retries; a well-formed action with a future date lands in the UI.
3. **J3** — An action where the change set is empty and the `actionType` is `NO_ACTION` when the request clearly demanded an action receives an eval score of 1 with a clear rationale; the UI flags the card.
4. **J4** — A context snapshot containing `alice@example.com`, `+1-555-0100`, and `123 Main St` is submitted; the LLM call log shows only `[REDACTED-EMAIL]`, `[REDACTED-PHONE]`, `[REDACTED-ADDRESS]`; the entity's `request.rawContext` retains the raw data for audit.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named personal-assistant demonstrating the single-agent × planning-travel cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-planning-travel-personal-assistant. Java package io.akka.samples.personalassistant.
Akka 3.6.0. HTTP port 9559.

Components to wire (exactly):

- 1 AutonomousAgent AssistantAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/assistant-agent.md>) and
  .capability(TaskAcceptance.of(HANDLE_REQUEST).maxIterationsPerTask(3)). The task receives
  the natural-language request as its instruction text and the sanitized context snapshot as a
  task ATTACHMENT (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes)
  is the canonical call). Output: AssistantAction{actionType: ActionType, explanation: String,
  changeSet: List<ChangeField>, decidedAt: Instant}. The agent is configured with a
  before-tool-call guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries within its
  3-iteration budget.

- 1 Workflow AssistantWorkflow per requestId with three steps:
  * awaitSanitizedStep — polls AssistantEntity.getState every 1s; on state.sanitized().isPresent()
    advances to actStep. WorkflowSettings.stepTimeout 15s (sanitizer is in-process and fast).
  * actStep — emits ActionStarted, then calls componentClient.forAutonomousAgent(
    AssistantAgent.class, "assistant-" + requestId).runSingleTask(
      TaskDef.instructions(state.request().get().naturalLanguageRequest())
        .attachment("context.json", state.sanitized().get().redactedSnapshot().getBytes())
    ) — returns a taskId, then forTask(taskId).result(HANDLE_REQUEST) to fetch the action.
    On success calls AssistantEntity.recordAction(action). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(2).failoverTo(AssistantWorkflow::error).
  * evalStep — runs a deterministic rule-based ActionScorer (NOT an LLM call) over the
    recorded action: checks that the changeSet is non-empty when actionType is not NO_ACTION,
    that every ChangeField has a non-empty newValue, that the explanation begins with a
    subject-verb phrase, and that the actionType matches what the changeSet implies (e.g.
    COMPLETE_TASK must have a completed field flip). Emits EvaluationScored{score: 1-5,
    rationale: String}. WorkflowSettings.stepTimeout 5s. error step transitions entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity AssistantEntity (one per requestId). State AssistantState{requestId:
  String, request: Optional<AssistantRequest>, sanitized: Optional<SanitizedContext>, action:
  Optional<AssistantAction>, eval: Optional<EvalResult>, status: RequestStatus, createdAt:
  Instant, finishedAt: Optional<Instant>}. RequestStatus enum: SUBMITTED, SANITIZED, ACTING,
  ACTION_RECORDED, EVALUATED, FAILED. Events: RequestSubmitted{request}, ContextSanitized{sanitized},
  ActionStarted{}, ActionRecorded{action}, EvaluationScored{eval}, RequestFailed{reason}.
  Commands: submit, attachSanitized, markActing, recordAction, recordEvaluation, fail, getState.
  emptyState() returns AssistantState.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...) inside
  the event-applier.

- 1 Consumer ContextSanitizer subscribed to AssistantEntity events; on RequestSubmitted runs
  a regex+heuristic redaction pipeline (emails, phone numbers, physical addresses, person
  names appearing in attendee lists and task descriptions, account-id-like tokens) over the
  serialized rawContext JSON, computes the list of categories found, builds SanitizedContext,
  then calls AssistantEntity.attachSanitized(sanitized). After attachSanitized lands, the same
  Consumer starts an AssistantWorkflow with id = "assistant-" + requestId.

- 1 View AssistantView with row type AssistantRow (mirrors AssistantState minus
  request.rawContext — the audit log keeps the raw; the view holds the sanitized form for
  the UI). Table updater consumes AssistantEntity events. ONE query getAllRequests:
  SELECT * AS requests FROM assistant_view. No WHERE status filter — Akka cannot auto-index
  enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * AssistantEndpoint at /api with POST /requests (body
    {naturalLanguageRequest, rawContext: {events: [...], tasks: [...], timezone}, submittedBy};
    mints requestId; calls AssistantEntity.submit; returns {requestId}), GET /requests (list
    from getAllRequests, sorted newest-first), GET /requests/{id} (one row), GET /requests/sse
    (Server-Sent Events forwarded from the view's stream-updates), and three /api/metadata/*
    endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- AssistantTasks.java declaring one Task<R> constant: HANDLE_REQUEST = Task.name("Handle
  assistant request").description("Read the attached context snapshot and produce an
  AssistantAction satisfying the user's natural-language request").resultConformsTo(
  AssistantAction.class). DO NOT skip this — the AutonomousAgent requires its companion
  Tasks class (Lesson 7).

- Domain records ContextEvent, ContextTask, Priority, ContextSnapshot, SanitizedContext,
  ChangeField, AssistantAction, ActionType, EvalResult, AssistantRequest, AssistantState,
  RequestStatus.

- WriteGuardrail.java implementing the before-tool-call hook. Reads the candidate tool-call
  arguments, runs the four checks listed in eval-matrix.yaml G1, and either passes the
  call through or returns Guardrail.reject(<structured-error>) to force the agent loop to
  retry.

- ActionScorer.java — pure deterministic logic (no LLM). Inputs: AssistantAction and the
  original natural-language request string. Outputs: EvalResult. Scoring rubric documented
  in Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9559 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The AssistantAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/context-profiles.jsonl with 3 seeded context profiles:
  a Travel week (5 events + 8 tasks including hotel check-in, airport transfer, packing
  reminder), a Day-trip (3 events + 4 tasks), and a Minimal profile (1 event + 2 tasks).
  Each profile contains 2–3 plausible PII strings (contact email, phone, address) so S1
  has work to do.

- src/main/resources/sample-events/seed-requests.jsonl with 6 seeded natural-language
  requests paired to the context profiles: e.g. "Schedule a reminder to pick up travel
  insurance docs on Thursday morning" → Travel week; "Mark the taxi booking task as
  complete" → Travel week; "Add a task to confirm the hotel's late checkout" → Day-trip.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, S1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = execute
  (the agent's action is applied directly to the in-process state — the user reviews the
  result in the UI), oversight.human_on_loop = true (the user reads the change-set and
  can undo), failure.failure_modes including "wrong-date-write", "cross-context-write",
  "pii-leakage-via-llm", "no-action-when-action-needed"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/assistant-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: PersonalAssistant", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of request cards; right = selected-request detail with original request, sanitized
  context preview, action type badge, explanation paragraph, change-set table, and eval-score
  chip). Browser title exactly: <title>Akka Sample: PersonalAssistant</title>. No subtitle
  on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent (see Mock LLM provider block below). Sets
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(requestId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    handle-request.json — 8 AssistantAction entries covering all ActionType values.
      Each entry has an explanation paragraph and a changeSet array with 1–3 ChangeField
      items. Each ChangeField has a non-empty newValue and an optional oldValue. ActionTypes
      vary: 2 CREATE_EVENT, 2 CREATE_TASK, 1 UPDATE_TASK, 1 COMPLETE_TASK, 1 SET_REMINDER,
      1 NO_ACTION. Plus 2 deliberately MALFORMED entries (one with a date in the past; one
      with an empty title) — the guardrail blocks both. The mock selects a malformed entry
      on the FIRST iteration of every 3rd request (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(requestId) helper makes per-request selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. AssistantAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion AssistantTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (actStep 60s,
  awaitSanitizedStep 15s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on AssistantState is Optional<T>. The view table
  updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: AssistantTasks.java with HANDLE_REQUEST = Task.name(...).description(...)
  .resultConformsTo(AssistantAction.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9559 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more. Any tab removed in an earlier
  iteration must be deleted from the HTML; display:none is not enough.
- The single-agent invariant: there is exactly ONE AutonomousAgent (AssistantAgent). The
  on-decision eval is rule-based (ActionScorer.java) and does NOT make an LLM call —
  keeping the pattern's "one agent" promise honest.
- The context snapshot is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated actStep uses TaskDef.attachment(...) and not string
  interpolation into the instruction text.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as an external check that runs after the agent returns. Lesson 1's AutonomousAgent
  contract is the authoritative reference for how the hook is registered.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block. Per
  Lesson 25, /akka:specify handles the key during generation.
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
