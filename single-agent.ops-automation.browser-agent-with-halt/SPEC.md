# SPEC — web-navigation-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** WebNavigationAgent.
**One-line pitch:** A user submits a task goal; one vision-language agent observes rendered page screenshots, selects browser actions (click, type, scroll, navigate), and drives a headless browser to completion — pausing for human approval on high-stakes actions and stopping immediately if an operator halt signal arrives.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the ops-automation domain. One `WebNavigatorAgent` (AutonomousAgent) carries the entire decision loop; it sees a screenshot and a goal, picks the next action, and the surrounding components execute it, audit it, and govern it. Three governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** (`ActionGuardrail`) runs before every browser action executes — checking the target domain against an allowed list, screening for forbidden-action types, and classifying whether the action is high-stakes. Blocked actions force the agent to retry with a different strategy rather than exiting the session.
- An **operator/regulator halt switch** (`HaltController` EventSourcedEntity) provides a global kill switch. The workflow checks the halt flag before each action step; if set, the session transitions to `HALTED` immediately and no further browser actions execute.
- A **human-in-the-loop gate** fires when the guardrail classifies an action as high-stakes (form submit, purchase confirmation, account deletion, password change). The workflow pauses at `AWAITING_APPROVAL`; a human approves or rejects via the API. Only on approval does the action execute.

The blueprint shows that autonomous browser agents can operate within explicit governance bounds: the agent is free within the permitted action space, but the system enforces who can stop it and what humans must see before irreversible actions land.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **Task goal** (e.g., "Go to the Akka docs site and find the latest SDK release version") and optionally fills **Starting URL** and **Max steps** (default 20).
2. The user clicks **Start navigation**. The UI POSTs to `/api/sessions` and receives a `sessionId`.
3. The session card appears in the live list in `STARTING` state. Within ~1 s, it transitions to `NAVIGATING`.
4. On each action step, the card shows the current URL, the latest screenshot thumbnail, and the action the agent just took. The action log accumulates in the right pane.
5. If a high-stakes action is pending, the card transitions to `AWAITING_APPROVAL`. The right pane shows the proposed action with its target URL and element description. The human clicks **Approve** or **Reject**.
6. On **Approve**, the action executes and navigation resumes. On **Reject**, the agent receives a rejection notice and must plan an alternative path.
7. When the agent decides the task is complete (or hits `maxSteps`), the card transitions to `COMPLETED`. The right pane shows the final screenshot, the task outcome string, and the full action log.
8. The operator can hit **Halt** at any time. The card immediately transitions to `HALTED` with no further actions.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `SessionEndpoint` | `HttpEndpoint` | `/api/sessions/*` — start, list, get, approve, reject, SSE; serves `/api/metadata/*`. | — | `SessionEntity`, `SessionView`, `HaltController` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `SessionEntity` | `EventSourcedEntity` | Per-session lifecycle: starting → navigating → awaiting-approval → completed/halted/rejected. Source of truth. | `SessionEndpoint`, `ScreenshotConsumer`, `NavigationWorkflow` | `SessionView` |
| `HaltController` | `EventSourcedEntity` | Global halt flag (one instance, id = "global"). Commands: `setHalt`, `clearHalt`, `getHalt`. Events: `HaltSet`, `HaltCleared`. | `SessionEndpoint` | `NavigationWorkflow` |
| `ScreenshotConsumer` | `Consumer` | Subscribes to `ActionExecuted` events; captures the post-action page screenshot via the headless browser; emits `ScreenshotCaptured` back to the entity. | `SessionEntity` events | `SessionEntity` |
| `NavigationWorkflow` | `Workflow` | One workflow per session. Steps: `initStep` → `actionStep` (looped) → `hitlStep` (conditional) → `completeStep`. Polls `HaltController` before each `actionStep`. | started by `SessionEndpoint` | `WebNavigatorAgent`, `SessionEntity`, `HaltController` |
| `WebNavigatorAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the task goal and current page screenshot as a task attachment; returns `BrowserAction`. | invoked by `NavigationWorkflow` | returns action |
| `ActionGuardrail` | supporting class | `before-tool-call` guardrail registered on `WebNavigatorAgent`. Checks domain allowlist, forbidden actions, and high-stakes classifier. | invoked per agent tool-call | returns allow / block / flag-hitl |
| `SessionView` | `View` | Read model: one row per session for the UI. | `SessionEntity` events | `SessionEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record TaskGoal(
    String goalId,
    String description,
    String startingUrl,
    int maxSteps,
    String submittedBy,
    Instant submittedAt
) {}

enum ActionType {
    CLICK, TYPE, SCROLL, NAVIGATE, SCREENSHOT, COMPLETE, REJECT_TASK
}

record BrowserAction(
    ActionType actionType,
    String targetSelector,   // CSS selector or descriptive label
    String targetUrl,        // for NAVIGATE actions
    String inputText,        // for TYPE actions
    String rationale,        // one sentence why this action
    boolean highStakes,      // agent-declared; guardrail may override
    Instant decidedAt
) {}

record ActionOutcome(
    String actionId,
    BrowserAction action,
    boolean executed,
    String blockedReason,    // null when executed = true
    String screenshotPath,   // path inside session snapshot store
    Instant executedAt
) {}

record HitlDecision(
    String decisionId,
    String actionId,
    HitlOutcome outcome,
    String reviewedBy,
    Instant decidedAt
) {}
enum HitlOutcome { APPROVED, REJECTED }

record SessionOutcome(
    boolean success,
    String taskResult,       // summary of what was found/done
    int stepsUsed,
    Instant completedAt
) {}

record Session(
    String sessionId,
    Optional<TaskGoal> goal,
    List<ActionOutcome> actionLog,
    Optional<BrowserAction> pendingAction,
    Optional<HitlDecision> lastHitlDecision,
    Optional<SessionOutcome> outcome,
    SessionStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SessionStatus {
    STARTING, NAVIGATING, AWAITING_APPROVAL, COMPLETED, HALTED, FAILED, REJECTED
}
```

Events on `SessionEntity`: `SessionStarted`, `ActionPlanned`, `ActionExecuted`, `ScreenshotCaptured`, `HitlRequested`, `HitlResolved`, `SessionCompleted`, `SessionHalted`, `SessionFailed`, `SessionRejected`.

Every nullable lifecycle field on the `Session` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/sessions` — body `{ description, startingUrl, maxSteps, submittedBy }` → `{ sessionId }`.
- `GET /api/sessions` — list all sessions, newest-first.
- `GET /api/sessions/{id}` — one session.
- `GET /api/sessions/sse` — Server-Sent Events; one event per state transition.
- `POST /api/sessions/{id}/approve` — body `{ reviewedBy }` → approves the pending HITL action.
- `POST /api/sessions/{id}/reject` — body `{ reviewedBy, reason }` → rejects the pending HITL action.
- `POST /api/halt` — body `{}` → sets the global halt flag immediately.
- `DELETE /api/halt` — clears the global halt flag.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Web Navigation Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of sessions (status pill + age + goal preview) and a right pane with the selected session's detail — current URL, latest screenshot, action log table, HITL approval panel (when status is `AWAITING_APPROVAL`), and outcome summary.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`ActionGuardrail`): runs before every `BrowserAction` the agent selects. Checks (1) target domain is in the configured allowed-domain list, (2) action type is not in the forbidden-action set, and (3) the action is not classified as high-stakes (form submit, checkout, account settings change, password input). On domain/forbidden failure returns a `block` decision — the agent loop receives a structured rejection and must plan a different action within its iteration budget. On high-stakes classification, returns a `flag-hitl` decision that causes the workflow to pause at `hitlStep` rather than executing.
- **H1 — operator/regulator halt** (`HaltController`): a global, immediately effective kill switch. The workflow reads `HaltController.getHalt()` before every `actionStep`. If the flag is set, the workflow transitions the session to `HALTED` without calling the agent or executing any further browser action. The halt can be set via `POST /api/halt` by any operator (or by an automated regulator monitor) and cleared via `DELETE /api/halt`. The flag persists across service restarts because `HaltController` is an EventSourcedEntity.
- **HITL1 — application HITL gate** (`hitlStep` in `NavigationWorkflow`): fires when `ActionGuardrail` returns `flag-hitl`. The workflow writes `HitlRequested` to `SessionEntity`, transitions the session to `AWAITING_APPROVAL`, and suspends. A human reviews the proposed action in the UI and calls `POST /api/sessions/{id}/approve` or `/reject`. On approve, the workflow executes the action and returns to `actionStep`. On reject, the agent receives a rejection message and must plan an alternative (consuming one iteration). If no decision arrives within the HITL timeout (configurable, default 300 s), the session transitions to `FAILED`.

## 9. Agent prompts

- `WebNavigatorAgent` → `prompts/web-navigator.md`. The single decision-making LLM. System prompt instructs it to read the current-page screenshot, understand the task goal, and return the single next `BrowserAction` as a typed JSON object.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a navigation task; within the step budget the session reaches `COMPLETED` with a full action log and final screenshot visible in the UI.
2. **J2** — Operator triggers `POST /api/halt` while the session is `NAVIGATING`; within one action step the session transitions to `HALTED` and no further browser actions execute.
3. **J3** — Agent selects a form-submit action; `ActionGuardrail` returns `flag-hitl`; session pauses at `AWAITING_APPROVAL`; human approves; navigation resumes and eventually reaches `COMPLETED`.
4. **J4** — Agent attempts to navigate to a domain not in the allowed list; `ActionGuardrail` blocks with a structured rejection; the agent retries with a different action on the same allowed domain.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named web-navigation-agent demonstrating the single-agent × ops-automation cell.
Runs out of the box (no external services other than the headless browser, which is bundled
in-process via Playwright for Java). Maven group io.akka.samples. Maven artifact
single-agent-ops-automation-browser-agent-with-halt. Java package
io.akka.samples.webnavigationagentwebvoyager. Akka 3.6.0. HTTP port 9797.

Components to wire (exactly):

- 1 AutonomousAgent WebNavigatorAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/web-navigator.md>) and
  .capability(TaskAcceptance.of(NavigationTasks.PLAN_NEXT_ACTION).maxIterationsPerTask(5)).
  The task receives the goal text and the current step index as the task instructions field;
  the current page screenshot (PNG bytes) is passed as a task ATTACHMENT named "screenshot.png"
  (TaskDef.attachment("screenshot.png", screenshotBytes) — NOT inlined as text).
  Output: BrowserAction{actionType: ActionType, targetSelector: String, targetUrl: String,
  inputText: String, rationale: String, highStakes: boolean, decidedAt: Instant}.
  The agent is configured with a before-tool-call guardrail (see G1 in eval-matrix.yaml)
  registered via the agent's guardrail-configuration block. On guardrail block, the agent loop
  retries the action plan within its 5-iteration budget.

- 1 Workflow NavigationWorkflow per sessionId with steps:
  * initStep — calls SessionEntity.markNavigating; captures the initial page screenshot
    (via headless browser at goal.startingUrl); calls SessionEntity.recordScreenshot.
    WorkflowSettings.stepTimeout 30s.
  * actionStep — checks HaltController.getHalt(); if halted, transitions to haltedStep.
    Otherwise calls componentClient.forAutonomousAgent(WebNavigatorAgent.class,
    "navigator-" + sessionId).runSingleTask(TaskDef.instructions(formatGoal(session))
    .attachment("screenshot.png", session.latestScreenshot)) to get a BrowserAction.
    If ActionGuardrail flagged the action as high-stakes, transitions to hitlStep.
    If blocked, records the block reason and loops back to actionStep (consuming one step).
    WorkflowSettings.stepTimeout 90s with defaultStepRecovery maxRetries(2)
    .failoverTo(NavigationWorkflow::failedStep).
  * hitlStep — writes HitlRequested to SessionEntity; suspends waiting for an external
    signal (SessionEndpoint calls NavigationWorkflow.resume(decision) on approve/reject).
    WorkflowSettings.stepTimeout 300s. On resume with APPROVED: execute the action and
    go back to actionStep. On REJECTED: send rejection notice and go back to actionStep
    (consuming one step).
  * completeStep — writes SessionCompleted{outcome} to SessionEntity.
    WorkflowSettings.stepTimeout 10s.
  * haltedStep — writes SessionHalted to SessionEntity. WorkflowSettings.stepTimeout 5s.
  * failedStep — writes SessionFailed{reason} to SessionEntity. WorkflowSettings.stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity SessionEntity (one per sessionId). State Session{sessionId: String,
  goal: Optional<TaskGoal>, actionLog: List<ActionOutcome>, pendingAction: Optional<BrowserAction>,
  lastHitlDecision: Optional<HitlDecision>, outcome: Optional<SessionOutcome>,
  status: SessionStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  SessionStatus enum: STARTING, NAVIGATING, AWAITING_APPROVAL, COMPLETED, HALTED, FAILED, REJECTED.
  Events: SessionStarted{goal}, ActionPlanned{action}, ActionExecuted{outcome},
  ScreenshotCaptured{screenshotPath}, HitlRequested{pendingAction}, HitlResolved{decision},
  SessionCompleted{outcome}, SessionHalted{}, SessionFailed{reason}, SessionRejected{reason}.
  Commands: start, markNavigating, recordAction, recordScreenshot, requestHitl, resolveHitl,
  complete, halt, fail, reject, getSession. emptyState() returns Session.initial("") with all
  Optional fields as Optional.empty() and status = STARTING (Lesson 3).

- 1 EventSourcedEntity HaltController (id = "global"). State HaltState{halted: boolean,
  setAt: Optional<Instant>, clearedAt: Optional<Instant>}. Events: HaltSet{setAt},
  HaltCleared{clearedAt}. Commands: setHalt, clearHalt, getHalt. emptyState() returns
  HaltState{halted: false, setAt: Optional.empty(), clearedAt: Optional.empty()}.

- 1 Consumer ScreenshotConsumer subscribed to SessionEntity events; on ActionExecuted
  event, uses the headless browser client to capture a screenshot of the current page state,
  writes the PNG to the session snapshot store (in-memory for dev mode), then calls
  SessionEntity.recordScreenshot(screenshotPath). The Consumer handles delivery-at-least-once
  by checking that the latest actionLog entry does not already have a screenshotPath before
  re-capturing.

- 1 View SessionView with row type SessionRow (mirrors Session minus the full actionLog bytes —
  the view holds the count and the latest action summary for the UI list; the full log is
  fetched on demand via GET /api/sessions/{id}). Table updater consumes SessionEntity events.
  ONE query: getAllSessions: SELECT * AS sessions FROM session_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * SessionEndpoint at /api with:
    POST /sessions (body {description, startingUrl, maxSteps, submittedBy}; mints sessionId;
    calls SessionEntity.start; starts NavigationWorkflow; returns {sessionId}),
    GET /sessions (list from getAllSessions, sorted newest-first),
    GET /sessions/{id} (one row),
    GET /sessions/sse (Server-Sent Events forwarded from the view stream-updates),
    POST /sessions/{id}/approve (body {reviewedBy}; calls NavigationWorkflow.resume(APPROVED);
    returns 200),
    POST /sessions/{id}/reject (body {reviewedBy, reason}; calls NavigationWorkflow.resume(REJECTED);
    returns 200),
    POST /halt (calls HaltController.setHalt; returns 200),
    DELETE /halt (calls HaltController.clearHalt; returns 200), and three
    /api/metadata/* endpoints serving YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- NavigationTasks.java declaring one Task<R> constant: PLAN_NEXT_ACTION =
  Task.name("Plan next browser action")
    .description("Observe the current page screenshot and task goal; return the single next BrowserAction")
    .resultConformsTo(BrowserAction.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records TaskGoal, BrowserAction, ActionType, ActionOutcome, HitlDecision, HitlOutcome,
  SessionOutcome, Session, SessionStatus.

- ActionGuardrail.java implementing the before-tool-call hook. Reads the candidate BrowserAction,
  runs: (1) domain allowlist check on targetUrl, (2) forbidden-action-type check on actionType,
  (3) high-stakes classifier (form-submit actions, NAVIGATE to /checkout, /account/settings,
  /password, /purchase). Returns Guardrail.block(reason) for domain/forbidden failures;
  returns Guardrail.flagHitl() for high-stakes; returns Guardrail.allow() otherwise.

- HeadlessBrowserClient.java — wraps Playwright for Java in a thin facade. Methods:
  navigateTo(url): void, captureScreenshot(): byte[], click(selector): void,
  type(selector, text): void, scroll(direction, amount): void. In dev mode (no real browser
  installed) falls back to returning a synthetic placeholder PNG so the workflow does not block.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9797 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The WebNavigatorAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-tasks.jsonl with 3 seeded task goals:
  (1) "Find the latest Akka SDK release version on doc.akka.io",
  (2) "Look up the current weather in San Francisco on a public weather site",
  (3) "Find the abstract of the paper 'WebVoyager: Building an End-to-End Web Agent with
  Large Multimodal Models' on a public paper repository."

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (G1, H1, HITL1) matching the
  mechanisms in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.capabilities including web-navigation,
  decisions.authority_level = fully-autonomous (agent acts without human unless high-stakes),
  oversight.human_on_loop = true (HITL gate fires for high-stakes), oversight.human_in_loop =
  false (operator is on-loop, not in-loop for routine actions), failure.failure_modes including
  "unintended-form-submission", "navigation-to-disallowed-domain", "runaway-session-no-halt",
  "hitl-timeout-session-stuck"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/web-navigator.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Web Navigation Agent (WebVoyager)",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of session cards; right = selected-session detail with current URL, latest
  screenshot thumbnail, action log, HITL approval panel when status = AWAITING_APPROVAL,
  outcome summary, global halt button).
  Browser title exactly: <title>Akka Sample: Web Navigation Agent</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct BrowserAction outputs per agent (see Mock LLM provider block below). Sets
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
  dispatch on the Task<R> id. The branch for PLAN_NEXT_ACTION reads
  src/main/resources/mock-responses/plan-next-action.json, picks one entry pseudo-randomly
  per call (seedFor(sessionId, stepIndex)), and deserialises into BrowserAction.
- Per-task mock-response shapes for THIS blueprint:
    plan-next-action.json — 10 BrowserAction entries covering multiple ActionType values
    (CLICK, TYPE, SCROLL, NAVIGATE, COMPLETE). Each entry has a non-empty targetSelector
    or targetUrl, a rationale sentence, and a highStakes flag (2 entries are highStakes=true
    to exercise the HITL path; 1 entry targets a domain outside the allowed list to exercise
    the guardrail-block path). The COMPLETE action type appears in 2 entries (one with a
    descriptive taskResult, one indicating the goal could not be reached within the budget).
- A MockModelProvider.seedFor(sessionId, stepIndex) helper makes per-step selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. WebNavigatorAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion NavigationTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (initStep 30s, actionStep 90s,
  hitlStep 300s, completeStep 10s, haltedStep 5s, failedStep 5s).
- Lesson 6: every nullable lifecycle field on the Session row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: NavigationTasks.java with PLAN_NEXT_ACTION = Task.name(...).description(...)
  .resultConformsTo(BrowserAction.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. Vision inputs require claude-sonnet-4-6 or gpt-4o (not older text-only
  models).
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9797 declared explicitly in application.conf's
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
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (WebNavigatorAgent).
  ActionGuardrail is a guardrail class, not an agent. HaltController and ScreenshotConsumer
  are infrastructure components. No second LLM call exists in this system.
- The screenshot is passed as a task ATTACHMENT, never inlined as base64 text in the
  instructions string. Verify the generated actionStep uses TaskDef.attachment("screenshot.png",
  screenshotBytes) and not string interpolation.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as an external check that runs after the agent returns.
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
