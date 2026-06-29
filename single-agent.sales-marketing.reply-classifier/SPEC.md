# SPEC — reply-classifier

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** ReplyClassifier.
**One-line pitch:** A user submits an inbound sales-email reply; one AI agent reads it (passed as a task attachment, never as inline prompt text) and returns a structured classification — `INTERESTED`, `NOT_INTERESTED`, `OBJECTION`, `OUT_OF_OFFICE`, or `UNSUBSCRIBE` — plus a proposed Pipedrive deal-stage update gated by a `before-tool-call` guardrail.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the sales-marketing domain. One `ReplyClassifierAgent` (AutonomousAgent) carries the entire classification decision; the surrounding components only prepare its input and audit its output. One governance mechanism is wired around the agent:

- A **before-tool-call guardrail** validates every proposed CRM mutation the agent emits before the call executes against Pipedrive. The guardrail checks that the proposed `newStage` is a valid next state given the deal's current stage, that the deal id in the tool call matches the reply's deal id, and that the `UNSUBSCRIBE` and `NOT_INTERESTED` classifications never trigger a stage-forward mutation. On rejection, the guardrail returns a structured error to the agent loop so it can correct the call or emit a `SKIP_CRM_UPDATE` signal within the same task.

The blueprint shows that a single-agent sales automation does not mean "unrestricted CRM writes" — one independent check sits between the agent's intent and the Pipedrive mutation.

## 3. User-facing flows

The user opens the App UI tab.

1. The user pastes an inbound email reply into the **Reply** textarea (or picks one of four seeded examples — an interested reply, a price-objection, an out-of-office auto-reply, and an unsubscribe request).
2. The user fills in a **Deal ID** (or accepts the seeded value) and a **Sender** field.
3. The user clicks **Classify reply**. The UI POSTs to `/api/replies` and receives a `replyId`.
4. The card appears in the live list in `RECEIVED` state. Within ~1 s, it transitions to `CLASSIFYING` as the workflow starts the agent task.
5. Within ~10–30 s, the agent returns. The card transitions to `CLASSIFIED`. The classification appears: an intent badge (`INTERESTED` / `NOT_INTERESTED` / `OBJECTION` / `OUT_OF_OFFICE` / `UNSUBSCRIBE`), a confidence score (0–100), a one-paragraph rationale, and the proposed CRM action.
6. If the proposed stage update passed the `before-tool-call` guardrail, the card shows `CRM_UPDATED` with the new stage name. If the guardrail rejected the mutation and the agent emitted `SKIP_CRM_UPDATE`, the card shows `CRM_SKIPPED` with the guardrail's rejection reason.
7. The user can submit another reply; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ReplyEndpoint` | `HttpEndpoint` | `/api/replies/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `ReplyEntity`, `ReplyView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ReplyEntity` | `EventSourcedEntity` | Per-reply lifecycle: received → classifying → classified → crm-updated / crm-skipped / failed. Source of truth. | `ReplyEndpoint`, `ClassificationWorkflow` | `ReplyView` |
| `ClassificationWorkflow` | `Workflow` | One workflow per reply. Steps: `classifyStep` → `crmUpdateStep`. | started by `ReplyEndpoint` after entity created | `ReplyClassifierAgent`, `ReplyEntity` |
| `ReplyClassifierAgent` | `AutonomousAgent` | The one decision-making LLM. Receives deal context in the task definition and the raw reply text as a task attachment; returns `ReplyClassification`. Proposes CRM mutations as tool calls gated by `CrmMutationGuardrail`. | invoked by `ClassificationWorkflow` | returns classification + optional CRM action |
| `CrmMutationGuardrail` | guardrail (supporting class) | `before-tool-call` hook on `ReplyClassifierAgent`. Validates every proposed `updateDealStage` tool call against the deal's current stage and the allowed-transitions map. | wired on `ReplyClassifierAgent` | allows/rejects tool call |
| `PipedriveClient` | supporting class | Simulated Pipedrive API client. In local dev, returns canned responses. A deployer replaces the base URL and supplies a real API token. | invoked by `ClassificationWorkflow.crmUpdateStep` | Pipedrive (or simulator) |
| `ReplyView` | `View` | Read model: one row per reply for the UI. | `ReplyEntity` events | `ReplyEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ReplySubmission(
    String replyId,
    String dealId,
    String sender,
    String rawReplyText,
    String subject,
    Instant receivedAt
) {}

record ReplyClassification(
    ReplyIntent intent,
    int confidenceScore,        // 0..100
    String rationale,
    CrmAction crmAction,
    Instant classifiedAt
) {}
enum ReplyIntent {
    INTERESTED, NOT_INTERESTED, OBJECTION, OUT_OF_OFFICE, UNSUBSCRIBE
}

record CrmAction(
    CrmActionType type,
    String newStage,            // nullable — populated only when type == UPDATE_STAGE
    String skipReason           // nullable — populated only when type == SKIP_CRM_UPDATE
) {}
enum CrmActionType { UPDATE_STAGE, SKIP_CRM_UPDATE }

record CrmUpdateResult(
    boolean success,
    String previousStage,
    String newStage,
    String guardrailRejectionReason,   // null when success == true
    Instant updatedAt
) {}

record Reply(
    String replyId,
    Optional<ReplySubmission> submission,
    Optional<ReplyClassification> classification,
    Optional<CrmUpdateResult> crmResult,
    ReplyStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ReplyStatus {
    RECEIVED, CLASSIFYING, CLASSIFIED, CRM_UPDATED, CRM_SKIPPED, FAILED
}
```

Events on `ReplyEntity`: `ReplyReceived`, `ClassificationStarted`, `ReplyClassified`, `CrmUpdateSucceeded`, `CrmUpdateSkipped`, `ReplyFailed`.

Every nullable lifecycle field on the `Reply` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/replies` — body `{ dealId, sender, rawReplyText, subject }` → `{ replyId }`.
- `GET /api/replies` — list all replies, newest-first.
- `GET /api/replies/{id}` — one reply.
- `GET /api/replies/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: ReplyClassifier</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted replies (status pill + intent badge + age) and a right pane with the selected reply's detail — raw reply text, classification rationale, CRM action outcome, and confidence score.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs on every `updateDealStage` tool call proposed by `ReplyClassifierAgent`. Checks: (1) the `dealId` in the tool call matches the `replyId`'s deal id; (2) the proposed `newStage` is in the allowed next-stage set for the deal's current stage; (3) the intent is not `UNSUBSCRIBE` or `NOT_INTERESTED` — those intents must never trigger a stage-forward mutation. On any failure, returns a structured rejection naming the failed check; the agent loop counts the rejection as one iteration and may retry with a corrected call or emit `SKIP_CRM_UPDATE`. Passing tool calls flow through to `ClassificationWorkflow.crmUpdateStep`.

## 9. Agent prompts

- `ReplyClassifierAgent` → `prompts/reply-classifier.md`. The single decision-making LLM. System prompt instructs it to read the attached raw reply, determine intent, assign a confidence score, write a rationale, and propose either an `updateDealStage` tool call or a `skipCrmUpdate` signal.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded `INTERESTED` reply; within 30 s the card shows `CRM_UPDATED` with the deal moved to `QUALIFIED`.
2. **J2** — The agent proposes a forbidden stage transition (mock LLM path produces an `INTERESTED` classification that attempts to advance a `CLOSED_WON` deal) — the `before-tool-call` guardrail rejects it; the agent retries and emits `SKIP_CRM_UPDATE`; the UI shows `CRM_SKIPPED` with the rejection reason.
3. **J3** — An `UNSUBSCRIBE` reply produces `CRM_SKIPPED` immediately — the guardrail fires on any stage-forward tool call the agent might propose for this intent.
4. **J4** — The raw reply text (including any personal email content) is preserved on the entity for audit but is never re-displayed in the UI list; only the intent badge and rationale are shown.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named reply-classifier demonstrating the single-agent × sales-marketing cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-sales-marketing-reply-classifier. Java package
io.akka.samples.qualifyrepliesfrompipedrivepersonswithai. Akka 3.6.0. HTTP port 9865.

Components to wire (exactly):

- 1 AutonomousAgent ReplyClassifierAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/reply-classifier.md>) and
  .capability(TaskAcceptance.of(CLASSIFY_REPLY).maxIterationsPerTask(3)). The task receives
  deal context (dealId, sender, subject, current stage) as its instruction text and the raw
  reply body as a task ATTACHMENT (NOT as inline prompt text —
  TaskDef.attachment("reply.txt", rawReplyText.getBytes()) is the canonical call).
  The agent exposes one tool: updateDealStage(dealId: String, newStage: String).
  The CrmMutationGuardrail is registered via the agent's before-tool-call guardrail-
  configuration block. On guardrail rejection the agent loop retries within its
  3-iteration budget.
  Output: ReplyClassification{intent: ReplyIntent, confidenceScore: int, rationale: String,
  crmAction: CrmAction, classifiedAt: Instant}.

- 1 Workflow ClassificationWorkflow per replyId with two steps:
  * classifyStep — emits ClassificationStarted, then calls
    componentClient.forAutonomousAgent(ReplyClassifierAgent.class, "classifier-" + replyId)
    .runSingleTask(
      TaskDef.instructions(formatDealContext(submission))
        .attachment("reply.txt", submission.rawReplyText().getBytes())
    ) — returns a taskId, then forTask(taskId).result(CLASSIFY_REPLY) to fetch the
    classification.
    On success calls ReplyEntity.recordClassification(classification).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(ClassificationWorkflow::error).
  * crmUpdateStep — reads classification.crmAction(). If type == UPDATE_STAGE, calls
    PipedriveClient.updateDealStage(dealId, classification.crmAction().newStage()) and
    calls ReplyEntity.recordCrmUpdate(result). If type == SKIP_CRM_UPDATE, calls
    ReplyEntity.skipCrmUpdate(skipReason). WorkflowSettings.stepTimeout 15s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ReplyEntity (one per replyId). State Reply{replyId: String,
  submission: Optional<ReplySubmission>, classification: Optional<ReplyClassification>,
  crmResult: Optional<CrmUpdateResult>, status: ReplyStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. ReplyStatus enum: RECEIVED, CLASSIFYING, CLASSIFIED,
  CRM_UPDATED, CRM_SKIPPED, FAILED. Events: ReplyReceived{submission},
  ClassificationStarted{}, ReplyClassified{classification}, CrmUpdateSucceeded{crmResult},
  CrmUpdateSkipped{skipReason}, ReplyFailed{reason}. Commands: submit, markClassifying,
  recordClassification, recordCrmUpdate, skipCrmUpdate, fail, getReply.
  emptyState() returns Reply.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...)
  inside the event-applier.

- 1 View ReplyView with row type ReplyRow (mirrors Reply minus submission.rawReplyText —
  the audit log keeps the raw text; the view holds the metadata for the UI). Table updater
  consumes ReplyEntity events. ONE query getAllReplies: SELECT * AS replies FROM reply_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * ReplyEndpoint at /api with POST /replies (body
    {dealId, sender, rawReplyText, subject};
    mints replyId; calls ReplyEntity.submit; starts ClassificationWorkflow; returns
    {replyId}), GET /replies (list from getAllReplies, sorted newest-first), GET /replies/{id}
    (one row), GET /replies/sse (Server-Sent Events forwarded from the view's
    stream-updates), and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ReplyTasks.java declaring one Task<R> constant: CLASSIFY_REPLY = Task.name("Classify
  reply").description("Read the attached email reply and return a ReplyClassification with
  intent, confidence, rationale, and a CRM action").resultConformsTo(ReplyClassification.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records ReplySubmission, ReplyClassification, ReplyIntent, CrmAction, CrmActionType,
  CrmUpdateResult, Reply, ReplyStatus.

- CrmMutationGuardrail.java implementing the before-tool-call hook. On each proposed
  updateDealStage tool call: (1) asserts dealId matches the submission's dealId, (2) checks
  the proposed newStage is in the allowed-transitions map for the deal's current stage,
  (3) rejects any stage-forward mutation when intent is UNSUBSCRIBE or NOT_INTERESTED.
  Returns Guardrail.reject(<structured-error>) on failure or passes through on success.

- PipedriveClient.java — a thin interface with one method updateDealStage(dealId, newStage).
  Default implementation is a simulator that returns canned CrmUpdateResult values. A deployer
  replaces the implementation with a real HTTP client and supplies the Pipedrive API token
  via ${?PIPEDRIVE_API_TOKEN} in application.conf.

- ALLOWED_STAGE_TRANSITIONS map (inside CrmMutationGuardrail or a companion
  StageTransitions.java): defines which newStage values are valid from each currentStage.
  Default map covers: PROSPECTING→{QUALIFIED}, QUALIFIED→{PROPOSAL,CLOSED_LOST},
  PROPOSAL→{CLOSED_WON,CLOSED_LOST}, CLOSED_WON→{}, CLOSED_LOST→{PROSPECTING}.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9865 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The ReplyClassifierAgent.definition()
  binds the configured provider via the per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-replies.jsonl with 4 seeded inbound reply examples:
  (1) an INTERESTED reply expressing readiness to schedule a demo, (2) a price-OBJECTION
  reply asking for a discount, (3) an OUT_OF_OFFICE auto-reply, (4) an UNSUBSCRIBE reply.
  Each carries a dealId, sender, subject, and rawReplyText.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC.

- risk-survey.yaml at the project root pre-filled for the sales-marketing domain.

- prompts/reply-classifier.md loaded as the agent system prompt.

- README.md at the project root as described in Section 1.

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of reply cards; right = selected-reply detail with raw reply text, intent badge,
  confidence score, rationale, and CRM action outcome).
  Browser title exactly: <title>Akka Sample: ReplyClassifier</title>. No subtitle on the
  Overview tab.

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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(replyId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    classify-reply.json — 8 ReplyClassification entries covering all 5 ReplyIntent values.
      Each entry has a rationale paragraph and a CrmAction. Valid entries: INTERESTED with
      UPDATE_STAGE to QUALIFIED; NOT_INTERESTED with SKIP_CRM_UPDATE; OBJECTION with
      SKIP_CRM_UPDATE (awaiting follow-up); OUT_OF_OFFICE with SKIP_CRM_UPDATE; UNSUBSCRIBE
      with SKIP_CRM_UPDATE. Plus 2 deliberately INVALID entries: one proposes UPDATE_STAGE
      with a newStage not in the allowed-transitions map; one proposes UPDATE_STAGE on a
      CLOSED_WON deal. The guardrail rejects both, exercising the retry path. The mock should
      select an invalid entry on the FIRST iteration of every 3rd reply (modulo seed) so J2
      is reproducible.
- A MockModelProvider.seedFor(replyId) helper makes per-reply selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ReplyClassifierAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ReplyTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (classifyStep
  60s, crmUpdateStep 15s, error 5s).
- Lesson 6: every nullable lifecycle field on the Reply row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: ReplyTasks.java with CLASSIFY_REPLY = Task.name(...).description(...)
  .resultConformsTo(ReplyClassification.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9865 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (ReplyClassifierAgent).
  The CRM update step is a workflow step calling PipedriveClient — it is NOT a second agent.
- The raw reply text is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated classifyStep uses TaskDef.attachment(...) and not
  string interpolation.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as an external check that runs after the agent returns.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
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
