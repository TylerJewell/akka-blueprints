# SPEC — persona-hot-reload

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** PersonaHotReload.
**One-line pitch:** An operator pushes updated persona keys (`agent_role`, `agent_goal`, `agent_instructions`, optionally `agent_model`) to a Configuration Store; the running agent adopts the new persona in-place — validated, logged, and monitored — without any service restart.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the governance-risk domain. One `PersonaAgent` (AutonomousAgent) carries every user-facing conversation; the surrounding components manage the live persona lifecycle — validating changes before they land, logging every transition, and monitoring agent behavior after a swap. Three governance mechanisms are wired around the agent:

- A **CI-side configuration gate** runs at write time: before any persona payload is written to the store, a `PersonaValidator` checks that the payload is structurally complete (all three mandatory keys present and non-empty), that `agent_instructions` does not contain forbidden directive patterns, and that `agent_model` (if present) is in the allowlisted model set. A rejected payload never reaches the agent.
- A **behavioral revalidation evaluator** runs immediately after each `PersonaActivated` event: a fixed probe set of N questions is sent through the newly-active agent, and the responses are scored against expected behavioral signatures. A probe failure is visible on the App UI and forwarded to the monitoring hook.
- A **human-on-the-loop monitoring hook** opens a post-swap observation window: for the first 5 minutes after each persona activation, all agent responses are forwarded to a configurable operator channel so a human can review live behavior without blocking the conversation flow.

The blueprint shows that a runtime-mutated agent can remain governed. Three independent checks sit across the swap event: one prevents bad configs from landing, one validates behavior after landing, and one gives a human operator visibility into the live window.

## 3. User-facing flows

The user opens the App UI tab.

1. The operator selects a persona fixture from the **Persona fixture** dropdown (Assistant, Escalation, Restricted) or pastes custom values into the three persona fields.
2. The operator clicks **Push persona**. The UI POSTs to `/api/personas/push` and receives a `changeId`.
3. A card appears in the **Persona change log** in `VALIDATING` state. Within ~1 s it transitions to `ACTIVE` (or `REJECTED` if the gate blocked it). The current-persona panel on the right updates immediately.
4. The user opens the **Chat** sub-panel on the App UI tab and sends a message. The response comes from the now-active persona.
5. Within ~10 s of activation, the `revalidationStep` completes. A revalidation badge appears on the card: PASS (all probes passed), PARTIAL (some probes passed), or FAIL (all probes failed). Failed probe details are listed below the badge.
6. For the first 5 minutes after each activation, an orange **Monitoring** banner is visible on the App UI. The banner shows elapsed time and the count of responses forwarded to the operator channel during that window.
7. The operator can push another persona at any time; a new card appears and the sequence restarts.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PersonaEndpoint` | `HttpEndpoint` | `/api/personas/*` — push, list, get, SSE, chat; serves `/api/metadata/*`. | — | `PersonaEntity`, `PersonaView`, `PersonaAgent` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `PersonaEntity` | `EventSourcedEntity` | Per-change lifecycle: validating → active or rejected; accumulates change log. Source of truth. | `PersonaEndpoint`, `PersonaValidator`, `PersonaWatchWorkflow` | `PersonaView` |
| `PersonaValidator` | `Consumer` | Subscribes to `PersonaChangeProposed` events; runs the gate checks; calls `PersonaEntity.activate` or `PersonaEntity.reject`. | `PersonaEntity` events | `PersonaEntity` |
| `PersonaWatchWorkflow` | `Workflow` | One workflow per `changeId`. Steps: `awaitValidationStep` → `activateAgentStep` → `revalidationStep` → `monitoringStep`. | started by `PersonaValidator` on activation | `PersonaAgent`, `PersonaEntity`, `BehavioralRevalidator` |
| `PersonaAgent` | `AutonomousAgent` | The one decision-making LLM. Its definition is rebuilt from the current `PersonaSnapshot` on each activation. Answers user chat queries. | invoked by `PersonaWatchWorkflow` (probes) and `PersonaEndpoint` (chat) | returns `AgentResponse` |
| `BehavioralRevalidator` | supporting class | Deterministic scorer. Sends fixed probes through the agent, compares responses to expected behavioral signatures. No LLM call. | invoked by `PersonaWatchWorkflow` | returns `RevalidationResult` |
| `PersonaView` | `View` | Read model: one row per persona change for the UI. | `PersonaEntity` events | `PersonaEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record PersonaPayload(
    String agentRole,
    String agentGoal,
    String agentInstructions,
    Optional<String> agentModel        // null means "keep current model"
) {}

record PersonaSnapshot(
    String changeId,
    PersonaPayload payload,
    String pushedBy,
    Instant pushedAt,
    Optional<String> rejectionReason
) {}

record ProbeResult(
    String probeId,
    String question,
    String response,
    ProbeOutcome outcome,              // PASS | FAIL | INCONCLUSIVE
    String rationale
) {}
enum ProbeOutcome { PASS, FAIL, INCONCLUSIVE }

record RevalidationResult(
    RevalidationStatus status,         // PASS | PARTIAL | FAIL
    List<ProbeResult> probeResults,
    Instant evaluatedAt
) {}
enum RevalidationStatus { PASS, PARTIAL, FAIL }

record MonitoringWindow(
    int forwardedCount,
    boolean open,
    Instant openedAt,
    Optional<Instant> closedAt
) {}

record AgentResponse(
    String changeId,
    String queryId,
    String responseText,
    Instant respondedAt
) {}

record PersonaChange(
    String changeId,
    Optional<PersonaSnapshot> snapshot,
    Optional<RevalidationResult> revalidation,
    Optional<MonitoringWindow> monitoring,
    PersonaChangeStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum PersonaChangeStatus {
    VALIDATING, REJECTED, ACTIVATING, ACTIVE, REVALIDATING, MONITORED, FAILED
}
```

Events on `PersonaEntity`: `PersonaChangeProposed`, `PersonaActivated`, `PersonaRejected`, `PersonaRevalidated`, `MonitoringWindowOpened`, `MonitoringWindowClosed`, `PersonaChangeFailed`.

Every nullable lifecycle field on the `PersonaChange` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/personas/push` — body `{ agentRole, agentGoal, agentInstructions, agentModel?, pushedBy }` → `{ changeId }`.
- `GET /api/personas` — list all persona changes, newest-first.
- `GET /api/personas/{id}` — one persona change.
- `GET /api/personas/current` — the currently active persona snapshot.
- `GET /api/personas/sse` — Server-Sent Events; one event per state transition.
- `POST /api/personas/chat` — body `{ queryId, message }` → `{ responseText, changeId }` (answered by active persona).
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: PersonaHotReload</title>`.

The App UI tab is a two-column layout: a left rail with the persona change log (status pill + revalidation badge + age) and a right pane with the selected change's detail — current persona fields, revalidation probe results, and monitoring window status. A Chat sub-panel below the detail panel lets the operator send a message and see the active persona's response in real time.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **C1 — Configuration gate** (`ci-gate`, applied inside `PersonaValidator` Consumer): checks every proposed persona payload for structural completeness, forbidden directive patterns, and model allowlist compliance before writing a `PersonaActivated` event. A blocked payload writes `PersonaRejected` instead; the agent never adopts the change.
- **E1 — Behavioral revalidation eval** (`eval-event`, `on-change-revalidation`): runs inside `PersonaWatchWorkflow.revalidationStep` immediately after each `PersonaActivated` event. A fixed probe set (N questions with expected behavioral signatures) is sent through `PersonaAgent`; `BehavioralRevalidator` scores the responses deterministically. Emits `PersonaRevalidated` with `RevalidationStatus` and per-probe detail. A FAIL result flags the UI card and is included in the monitoring-hook payload.
- **H1 — Post-swap monitoring window** (`hotl`, `deployer-runtime-monitoring`): runs inside `PersonaWatchWorkflow.monitoringStep` for 5 minutes after each activation. Every `AgentResponse` emitted during the window is forwarded to the operator channel (configurable via `application.conf` `monitoring.operator-webhook`). Emits `MonitoringWindowClosed` when the window expires.

## 9. Agent prompts

- `PersonaAgent` → `prompts/persona-agent.md`. The single decision-making LLM. System prompt is assembled at runtime from the active `PersonaSnapshot.payload` fields (`agentRole`, `agentGoal`, `agentInstructions`). The base prompt file provides structural framing and output constraints that remain constant across persona swaps.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Operator pushes the "Assistant" fixture persona; within 30 s the card reaches `MONITORED` with PASS revalidation; a chat query returns a response that matches the new role.
2. **J2** — Operator pushes a payload missing `agentGoal`; the gate rejects it; the card shows `REJECTED` with the missing-field reason; the previously active persona remains unchanged.
3. **J3** — Operator pushes the "Restricted" fixture persona whose revalidation probes include a question the restricted persona must refuse; a FAIL probe causes the card to show a PARTIAL or FAIL revalidation badge.
4. **J4** — A persona push with `agentModel = "gpt-4o"` is accepted; the agent answers the next chat query with the new model; the change log shows the previous and new model names.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named persona-hot-reload demonstrating the single-agent × governance-risk cell.
Requires a Redis-compatible instance or falls back to an in-process stub. Maven group io.akka.samples.
Maven artifact single-agent-governance-risk-akka-persona-hot-reload.
Java package io.akka.samples.hotreloadagentpersonaviaconfigurationstore. Akka 3.6.0. HTTP port 9799.

Components to wire (exactly):

- 1 AutonomousAgent PersonaAgent — definition() is rebuilt from the current PersonaSnapshot on each
  call to rebuildDefinition(snapshot). Returns AgentDefinition with
  .instructions(buildSystemPrompt(snapshot)) and
  .capability(TaskAcceptance.of(ANSWER_QUERY).maxIterationsPerTask(2)). The system prompt is
  assembled by concatenating the base framing from prompts/persona-agent.md with the snapshot's
  agentRole, agentGoal, and agentInstructions. If agentModel is present, .modelProvider(agentModel)
  is set; otherwise the configured default provider is used.
  Companion PersonaTasks.java declares: ANSWER_QUERY = Task.name("Answer query").description(
  "Answer the user message using the active persona").resultConformsTo(AgentResponse.class).
  DO NOT skip PersonaTasks — AutonomousAgent requires its companion Tasks class (Lesson 7).

- 1 Workflow PersonaWatchWorkflow per changeId with four steps:
  * awaitValidationStep — polls PersonaEntity.getChange every 1s; on change.status == ACTIVATING
    advances to activateAgentStep. WorkflowSettings.stepTimeout 10s.
  * activateAgentStep — calls PersonaAgent.rebuildDefinition(snapshot); emits PersonaActivated;
    calls PersonaEntity.markActive(changeId). WorkflowSettings.stepTimeout 10s with
    defaultStepRecovery maxRetries(1).failoverTo(PersonaWatchWorkflow::error).
  * revalidationStep — instantiates BehavioralRevalidator, runs all probes against PersonaAgent
    via componentClient.forAutonomousAgent(PersonaAgent.class, "probe-" + changeId)
    .runSingleTask(TaskDef.instructions(probe.question)); collects ProbeResult list;
    computes RevalidationStatus (PASS if all pass, PARTIAL if any pass, FAIL if none pass);
    calls PersonaEntity.recordRevalidation(result). WorkflowSettings.stepTimeout 60s with
    defaultStepRecovery maxRetries(1).failoverTo(PersonaWatchWorkflow::error).
  * monitoringStep — emits MonitoringWindowOpened; waits 300s (5 minutes); emits
    MonitoringWindowClosed with forwardedCount from the monitoring hook tally. WorkflowSettings
    .stepTimeout 320s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity PersonaEntity (one per changeId). State PersonaChange{changeId: String,
  snapshot: Optional<PersonaSnapshot>, revalidation: Optional<RevalidationResult>,
  monitoring: Optional<MonitoringWindow>, status: PersonaChangeStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. PersonaChangeStatus enum: VALIDATING, REJECTED, ACTIVATING,
  ACTIVE, REVALIDATING, MONITORED, FAILED. Events: PersonaChangeProposed{snapshot},
  PersonaActivated{snapshot}, PersonaRejected{reason}, PersonaRevalidated{result},
  MonitoringWindowOpened{openedAt}, MonitoringWindowClosed{forwardedCount},
  PersonaChangeFailed{reason}. Commands: propose, activate, reject, markActive, recordRevalidation,
  openMonitoringWindow, closeMonitoringWindow, fail, getChange. emptyState() returns
  PersonaChange.initial("") with no commandContext() reference (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer PersonaChangeConsumer subscribed to PersonaEntity events; on PersonaActivated
  opens the monitoring window: for the next 300s any AgentResponse emitted by PersonaAgent with
  matching changeId is forwarded to the operator webhook configured at
  monitoring.operator-webhook in application.conf (no-op if not set). Tracks forwardedCount.
  Calls PersonaEntity.closeMonitoringWindow(forwardedCount) after the window expires.
  Also: on PersonaChangeProposed calls PersonaValidator.validate(snapshot) synchronously and
  either calls PersonaEntity.activate(changeId) or PersonaEntity.reject(changeId, reason).

- 1 Consumer PersonaValidator (invoked inline by PersonaChangeConsumer) — runs three gate checks:
  (1) structural completeness: agentRole, agentGoal, agentInstructions all non-blank;
  (2) forbidden patterns: agentInstructions does not match a configurable regex list
      (default: "ignore previous instructions", "jailbreak", "DAN", "disregard your"); 
  (3) model allowlist: if agentModel is present, it must be in the set {claude-sonnet-4-6,
      claude-opus-4-5, gpt-4o, gpt-4o-mini, gemini-2.5-flash}. Returns ValidationResult{valid,
      rejectionReason}. This is NOT a Consumer Akka primitive — it is a plain Java class called
      from PersonaChangeConsumer. Name it PersonaValidator.java.

- 1 View PersonaView with row type PersonaChangeRow (mirrors PersonaChange). Table updater
  consumes PersonaEntity events. ONE query getAllChanges: SELECT * AS changes FROM persona_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * PersonaEndpoint at /api with POST /personas/push (body {agentRole, agentGoal,
    agentInstructions, agentModel?, pushedBy}; mints changeId; calls PersonaEntity.propose;
    returns {changeId}), GET /personas (list from getAllChanges, sorted newest-first), GET
    /personas/{id} (one row), GET /personas/current (the most-recent MONITORED or ACTIVE row),
    GET /personas/sse (Server-Sent Events forwarded from the view's stream-updates), POST
    /personas/chat (body {queryId, message}; routes to PersonaAgent.runSingleTask(ANSWER_QUERY,
    message); returns {responseText, changeId}), and three /api/metadata/* endpoints serving
    YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- PersonaTasks.java declaring one Task<R> constant: ANSWER_QUERY = Task.name("Answer query")
  .description("Answer the user message using the active persona").resultConformsTo(
  AgentResponse.class). DO NOT skip this — the AutonomousAgent requires its companion Tasks
  class (Lesson 7).

- Domain records PersonaPayload, PersonaSnapshot, ProbeResult, RevalidationResult,
  MonitoringWindow, AgentResponse, PersonaChange, PersonaChangeStatus, ProbeOutcome,
  RevalidationStatus, ValidationResult.

- BehavioralRevalidator.java — pure deterministic logic (no LLM). Inputs: PersonaSnapshot and
  a List<ProbeResult>. Outputs: RevalidationResult. Probe set loaded from
  src/main/resources/sample-events/probe-set.jsonl (3 seeded probes for each of the 3 persona
  fixtures). Scoring: if all ProbeOutcome == PASS → PASS; if any PASS → PARTIAL; else FAIL.
  Rationale documented in Javadoc.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9799 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. Also monitoring.operator-webhook = ${?MONITORING_WEBHOOK_URL}
  (optional, no-op if absent). Also redis.url = ${?REDIS_URL} with fallback "stub" triggering
  the in-process stub.

- src/main/resources/sample-events/persona-fixtures.jsonl with 3 seeded persona definitions:
  (a) "Assistant" — a general-purpose helpful assistant persona with moderate instructions;
  (b) "Escalation" — a persona focused on escalating unresolved issues to human agents;
  (c) "Restricted" — a persona with narrow scope (read-only answers, no advice, no speculation).

- src/main/resources/sample-events/probe-set.jsonl with N probe definitions per fixture:
  each probe has probeId, targetFixture, question, and expectedBehaviorSignature. The
  "Restricted" fixture probes include a question the restricted persona must refuse; this is
  the probe that should yield FAIL for J3.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies for
  the metadata endpoint to serve from classpath).

- eval-matrix.yaml at project root with 3 controls (C1, E1, H1). Matching simplified_view list.
  No regulation_anchors.

- risk-survey.yaml at project root with sector = governance-risk, decisions.authority_level =
  direct-action (the persona change directly affects agent behavior, not advisory),
  oversight.human_in_loop = false, oversight.human_on_loop = true, failure.failure_modes
  including "persona-injection-via-store", "model-swap-to-non-allowlisted-model",
  "revalidation-false-positive", "monitoring-window-missed"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/persona-agent.md loaded as the base agent system prompt framing.

- README.md at the project root: title "Akka Sample: Hot-Reload Agent Persona via Configuration
  Store", prerequisites (including Redis note), generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration section.
  NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no npm).
  Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left = persona
  change log + Chat sub-panel; right = selected-change detail with persona fields, revalidation
  probe table, monitoring window status). Browser title exactly:
  <title>Akka Sample: PersonaHotReload</title>. No subtitle on Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java per task dispatch. Per-task mock-response shapes:
    answer-query.json — 6 AgentResponse entries, 2 per fixture (Assistant, Escalation, Restricted).
    Restricted entries include one that simulates a refusal for the probe designed to fail (J3).
- MockModelProvider.seedFor(changeId) makes per-change selection deterministic.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:

- Lesson 1: PersonaAgent extends akka.javasdk.agent.autonomous.AutonomousAgent. PersonaTasks.java
  MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (awaitValidationStep 10s,
  activateAgentStep 10s, revalidationStep 60s, monitoringStep 320s, error 5s).
- Lesson 6: every nullable lifecycle field on PersonaChange is Optional<T>.
- Lesson 7: PersonaTasks.java with ANSWER_QUERY is mandatory.
- Lesson 8: model names — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated model ids.
- Lesson 9: run command is "/akka:build".
- Lesson 10: port 9799 in akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears user-facing.
- Lesson 12: UI fits 1080px content column.
- Lesson 13: integration label "Requires Redis (or in-process stub)" — no tier codes in any
  user-visible string.
- Lesson 23: no forbidden words.
- Lesson 24: static-resources/index.html includes mermaid CSS overrides and themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching by data-tab / data-panel attribute; exactly five tab-panel sections.
- Single-agent invariant: exactly ONE AutonomousAgent (PersonaAgent). BehavioralRevalidator
  is deterministic and does NOT make an LLM call.
- The persona definition is rebuilt from PersonaSnapshot — NOT from inline string interpolation
  into the task instructions at call time.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key, unrecoverable compile error, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
