# SPEC — bidi-demo

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** BidiDemo.
**One-line pitch:** A user opens a persistent channel, sends messages one at a time, and receives streamed response frames from one AI agent — with the channel staying open across turns until the client closes it or the turn budget is exhausted.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern applied to bidirectional streaming. One `ChannelAgent` (AutonomousAgent) handles every inbound message on a channel; surrounding components manage channel lifecycle, fan the message into a workflow, and stream frames back. One governance mechanism is wired around the agent:

- A **before-agent-response guardrail** validates every `ResponseFrame` before it hits the SSE wire: the frame must have a non-empty `content` string (unless `done == true`), the `turnId` must match the current turn, and the `frameIndex` must be monotonically increasing. A malformed frame triggers a retry inside the same task.

The blueprint shows streaming response composition — the agent does not return a single string but a `List<ResponseFrame>`, each frame delivered to the client as an SSE event as soon as the workflow publishes it.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **channel name** (or accepts the auto-generated one) and clicks **Open channel**. The UI POSTs to `/api/channels` and the channel card appears in `OPEN` state.
2. The user types a message in the channel's **input box** and clicks **Send**. The UI POSTs to `/api/channels/{id}/messages`.
3. Within ~1 s the message card transitions to `PROCESSING`. The SSE stream starts emitting `frame` events.
4. Response frames appear one by one in the right pane — each frame shows its `frameIndex`, the `content` text, and a small spinner until `done == true`.
5. When the `done` frame arrives, the turn card shows `COMPLETE`. The input box re-enables; the user can send another message.
6. At turn budget exhaustion (default 10 turns per channel) the channel enters `CLOSED` state. The input box locks. The user can open a new channel.
7. The user can observe a second browser tab subscribing to the same channel's SSE stream and seeing the same frames in real time.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ChannelEndpoint` | `HttpEndpoint` | `/api/channels/*` — open, send, list, get, SSE; serves `/api/metadata/*`. | — | `ChannelEntity`, `ChannelView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ChannelEntity` | `EventSourcedEntity` | Per-channel lifecycle: open → active → closed. Holds messages and response frames. | `ChannelEndpoint`, `MessageForwarder`, `ChannelWorkflow` | `ChannelView` |
| `MessageForwarder` | `Consumer` | Subscribes to `MessageReceived` events; starts a `ChannelWorkflow` run per message. | `ChannelEntity` events | `ChannelWorkflow` |
| `ChannelWorkflow` | `Workflow` | One workflow run per inbound message. Steps: `awaitTurnStep` → `respondStep` → `flushStep`. | started by `MessageForwarder` | `ChannelAgent`, `ChannelEntity` |
| `ChannelAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the channel history as a task attachment and the new message as task instructions; returns `List<ResponseFrame>`. | invoked by `ChannelWorkflow` | returns frame list |
| `FrameGuardrail` | guardrail class | Validates every `ResponseFrame` before it reaches the SSE wire: non-empty content (unless done), turnId match, monotonic frameIndex. | `ChannelAgent` before-response hook | pass/reject |
| `ChannelView` | `View` | Read model: one row per channel for the UI. | `ChannelEntity` events | `ChannelEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ChannelMessage(
    String messageId,
    String channelId,
    String content,
    String sentBy,
    Instant sentAt
) {}

record ResponseFrame(
    String turnId,
    int frameIndex,
    String content,   // empty string allowed only when done == true
    boolean done,
    Instant emittedAt
) {}

record Turn(
    String turnId,
    ChannelMessage message,
    List<ResponseFrame> frames,
    TurnStatus status,
    Instant startedAt,
    Optional<Instant> completedAt
) {}
enum TurnStatus { PROCESSING, COMPLETE, FAILED }

record Channel(
    String channelId,
    String channelName,
    List<Turn> turns,
    ChannelStatus status,
    int turnBudget,          // default 10
    Instant openedAt,
    Optional<Instant> closedAt
) {}

enum ChannelStatus { OPEN, ACTIVE, CLOSED, FAILED }
```

Events on `ChannelEntity`: `ChannelOpened`, `MessageReceived`, `TurnStarted`, `FramePublished`, `TurnCompleted`, `TurnFailed`, `ChannelClosed`.

Every nullable lifecycle field on `Channel` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/channels` — body `{ channelName, turnBudget?, openedBy }` → `{ channelId }`.
- `POST /api/channels/{id}/messages` — body `{ content, sentBy }` → `{ messageId, turnId }`.
- `GET /api/channels` — list all channels, newest-first.
- `GET /api/channels/{id}` — one channel with full turn and frame history.
- `GET /api/channels/{id}/sse` — Server-Sent Events; one event per frame plus lifecycle transitions.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: BidiDemo</title>`.

The App UI tab is a two-column layout: a left rail with the channel list (status pill + turn count + name + age) and a right pane showing the selected channel's full turn history — each turn card shows the inbound message, then the streamed response frames appearing one by one, then a turn-status badge.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. This is a baseline sample with no regulatory controls. The generated system wires one structural mechanism:

- **G1 — before-agent-response guardrail**: runs on every turn of `ChannelAgent`. Asserts the candidate frame list is non-empty, every frame has `content` non-empty unless it is the final `done == true` frame, every frame's `turnId` matches the current turn id, and `frameIndex` values are 0-based monotonically increasing with no gaps. On failure, returns a structured rejection to the agent loop and the task retries within its iteration budget.

## 9. Agent prompts

- `ChannelAgent` → `prompts/channel-agent.md`. The single conversational LLM. System prompt instructs it to read the channel history (attached as `history.json`) and the new user message (from task instructions), then return a list of `ResponseFrame` objects — one frame per sentence — ending with a frame whose `done` field is `true`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User opens a channel, sends one message; within 15 s all frames stream into the UI and the turn reaches `COMPLETE`.
2. **J2** — The agent's first iteration on a mock run returns a frame with an empty `content` and `done == false`; the `before-agent-response` guardrail rejects it; the second iteration produces valid frames; the UI shows only the valid frames.
3. **J3** — A channel that hits its turn budget (10 turns) enters `CLOSED`; the API rejects further messages with `409 Conflict`.
4. **J4** — Two browser tabs subscribe to the same channel's SSE stream; both receive the same frames in the same order with no duplication.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named bidi-demo demonstrating the single-agent × general-bidirectional-streaming
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-bidi-streaming-demo. Java package io.akka.samples.bididemo. Akka 3.6.0.
HTTP port 9799.

Components to wire (exactly):

- 1 AutonomousAgent ChannelAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/channel-agent.md>) and
  .capability(TaskAcceptance.of(RESPOND_TO_MESSAGE).maxIterationsPerTask(3)). The task receives
  the new user message as task instructions text and the channel turn history as a task ATTACHMENT
  named "history.json" (NOT as inline prompt text — Akka's TaskDef.attachment(name, bytes) is the
  canonical call). Output: List<ResponseFrame>{turnId, frameIndex, content, done, emittedAt}.
  The agent is configured with a before-agent-response guardrail (see G1 in eval-matrix.yaml)
  registered via the agent's guardrail-configuration block. On guardrail rejection the agent loop
  retries within its 3-iteration budget.

- 1 Workflow ChannelWorkflow per (channelId + turnId) with three steps:
  * awaitTurnStep — polls ChannelEntity.getChannel every 500ms; on
    channel.turns().stream().anyMatch(t -> t.turnId().equals(turnId) && t.status() == PROCESSING)
    advances to respondStep. WorkflowSettings.stepTimeout 10s.
  * respondStep — emits TurnStarted, then calls componentClient.forAutonomousAgent(
    ChannelAgent.class, "agent-" + channelId).runSingleTask(
      TaskDef.instructions("User says: " + message.content())
        .attachment("history.json", serializeTurnHistory(channel.turns()).getBytes())
    ) — returns taskId; then forTask(taskId).result(RESPOND_TO_MESSAGE) to fetch the frame list.
    Iterates the frame list and calls ChannelEntity.publishFrame(frame) for each. On success calls
    ChannelEntity.completeTurn(turnId). WorkflowSettings.stepTimeout 60s with
    defaultStepRecovery maxRetries(2).failoverTo(ChannelWorkflow::error).
  * flushStep — checks if the channel's turn count >= turnBudget; if so calls
    ChannelEntity.closeChannel("turn-budget-exhausted"). WorkflowSettings.stepTimeout 5s.
  error step calls ChannelEntity.failTurn(turnId, reason) then transitions to done.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ChannelEntity (one per channelId). State Channel{channelId: String,
  channelName: String, turns: List<Turn>, status: ChannelStatus, turnBudget: int,
  openedAt: Instant, closedAt: Optional<Instant>}. ChannelStatus enum: OPEN, ACTIVE, CLOSED,
  FAILED. Turn: {turnId, message: ChannelMessage, frames: List<ResponseFrame>, status: TurnStatus,
  startedAt, completedAt: Optional<Instant>}. TurnStatus: PROCESSING, COMPLETE, FAILED.
  Events: ChannelOpened{channelId, channelName, turnBudget, openedBy, openedAt},
  MessageReceived{messageId, channelId, content, sentBy, sentAt},
  TurnStarted{turnId, messageId},
  FramePublished{turnId, frame: ResponseFrame},
  TurnCompleted{turnId, completedAt},
  TurnFailed{turnId, reason},
  ChannelClosed{reason, closedAt}.
  Commands: open, receiveMessage, startTurn, publishFrame, completeTurn, failTurn, closeChannel,
  getChannel. emptyState() returns Channel.initial("") with empty turns list and
  status = OPEN (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state.

- 1 Consumer MessageForwarder subscribed to ChannelEntity events; on MessageReceived
  mints a turnId = "turn-" + messageId, calls ChannelEntity.startTurn(turnId, messageId), then
  starts a ChannelWorkflow with id = "workflow-" + turnId.

- 1 View ChannelView with row type ChannelRow (mirrors Channel minus the full frame content
  inside turns — turns are summarised as TurnSummary{turnId, status, frameCount,
  completedAt: Optional<Instant>} for the list view). Table updater consumes ChannelEntity events.
  ONE query getAllChannels: SELECT * AS channels FROM channel_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ChannelEndpoint at /api with POST /channels (body {channelName, turnBudget?, openedBy};
    mints channelId; calls ChannelEntity.open; returns {channelId}), POST /channels/{id}/messages
    (body {content, sentBy}; mints messageId; calls ChannelEntity.receiveMessage; returns
    {messageId, turnId}; rejects with 409 if channel.status != OPEN or ACTIVE), GET /channels
    (list from getAllChannels newest-first), GET /channels/{id} (full channel state), GET
    /channels/{id}/sse (Server-Sent Events stream of FramePublished and lifecycle events from the
    entity's event stream), and three /api/metadata/* endpoints serving YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ChannelTasks.java declaring one Task<R> constant: RESPOND_TO_MESSAGE = Task.name("Respond to
  message").description("Read the channel history attachment and the new user message, then return
  a list of ResponseFrame objects ending with done==true").resultConformsTo(ResponseFrameList.class).
  ResponseFrameList is a simple wrapper record: record ResponseFrameList(List<ResponseFrame> frames).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records ChannelMessage, ResponseFrame, Turn, TurnStatus, Channel, ChannelStatus,
  TurnSummary, ChannelRow, ResponseFrameList.

- FrameGuardrail.java implementing the before-agent-response hook. Reads the candidate
  ResponseFrameList from the LLM response and runs: (1) list is non-empty, (2) every frame except
  the last has non-empty content, (3) last frame has done == true, (4) frameIndex values are
  0-based with no gaps, (5) every frame's turnId matches the expected turnId passed in via the
  guardrail context. Returns Guardrail.reject(<structured-error>) on any failure.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9799 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/seed-conversations.jsonl with 3 seeded conversation scripts:
  a 3-turn general Q&A session, a 5-turn technical troubleshooting session, and a 2-turn creative
  brainstorm. Each script carries the user messages only; the agent responses are generated live
  (or mocked). Used to pre-populate the UI "Load example" buttons.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching Section 8 of this SPEC.
  No regulation_anchors — this is a general baseline, not a regulated domain.

- risk-survey.yaml at the project root pre-filled for a general streaming demo: data.pii = false,
  decisions.authority_level = interactive (agent responds to user queries in real time),
  oversight.human_in_loop = true (the user reads every frame), failure.failure_modes including
  "incomplete-frame-list", "malformed-done-flag", "turn-budget-miscalculation"; deployer fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/channel-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: BidiDemo", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no npm).
  Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left = channel
  list + new-channel button; right = selected channel's turn history, input box at bottom).
  Browser title exactly: <title>Akka Sample: BidiDemo</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java with per-task dispatch on Task<R> id.
- Per-task mock-response shapes for THIS blueprint:
    respond-to-message.json — 6 ResponseFrameList entries. Each has 3–5 frames with varied
    content (one sentence per frame), monotonically increasing frameIndex, correct turnId
    placeholder ("__TURN_ID__" replaced at selection time), and the last frame has done==true.
    Plus 2 deliberately MALFORMED entries — one with a frame having empty content and done==false,
    one with a non-sequential frameIndex — so G1 has work to do on every 3rd call.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ChannelAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ChannelTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (awaitTurnStep 10s, respondStep 60s,
  flushStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on Channel is Optional<T>.
- Lesson 7: ChannelTasks.java with RESPOND_TO_MESSAGE is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9799 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible string.
- Lesson 23: no forbidden words in narrative prose.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList index.
  The DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (ChannelAgent). The turn-budget
  checker in flushStep is pure deterministic logic — not an LLM call.
- The channel history is passed as a Task ATTACHMENT, never inlined into the agent's instructions.
- The before-agent-response guardrail (FrameGuardrail) is wired via the agent's
  guardrail-configuration block, not as a post-return check.
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
