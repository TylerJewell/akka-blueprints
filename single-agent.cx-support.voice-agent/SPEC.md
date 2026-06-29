# SPEC — voice-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** VoiceAgent.
**One-line pitch:** A caller speaks; one AI agent reads the sanitized transcript (passed as a task attachment, never as inline prompt text) and returns a typed reply that is synthesised into audio and played back — with a `before-agent-response` guardrail catching disallowed output before any TTS call fires.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the cx-support domain. One `VoiceConversationAgent` (AutonomousAgent) carries the entire response decision; the surrounding components prepare its input, govern its output, and maintain the conversation record. Two governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw transcript and the agent call — so the model never sees caller names, account numbers, phone numbers, or payment details in raw form.
- A **before-agent-response guardrail** validates the agent's reply on every turn: well-formed JSON, response text within the allowed character limit, no disallowed phrases (competitor names, promises of service unavailable to this tier, scripted escalation bypasses). A rejected reply triggers a retry inside the same task.

The blueprint shows that a voice-facing agent can enforce the same governance rigour as a text-only agent. The PII sanitizer and the output guardrail sit on either side of the one LLM call; the TTS stub only fires after the guardrail passes.

## 3. User-facing flows

The user opens the App UI tab.

1. The user picks a **conversation scenario** from a dropdown (billing query, technical support, cancellation request) or types a custom transcript directly.
2. The user enters a **caller identifier** and clicks **Send audio**. The UI POSTs to `/api/conversations` (new session) or `/api/conversations/{id}/turns` (existing session) and receives a `conversationId` and `turnId`.
3. The new session card appears in the live list in `AUDIO_RECEIVED` state. Within ~1 s it transitions to `TRANSCRIPT_SANITIZED` — the redacted transcript is visible with a small list of PII categories the sanitizer found.
4. Within ~10–30 s the workflow's `respondStep` completes. The card transitions to `RESPONDING` then `REPLY_RECORDED`. The reply appears: the agent's text response, a voice-tone badge (WARM / NEUTRAL / FIRM), and a topic-classification chip.
5. Within ~1 s of `REPLY_RECORDED`, the `synthesiseStep` finishes. The card reaches `AUDIO_SYNTHESISED` and shows a "Play audio" button (the stub returns a short beep WAV in non-real-LLM mode).
6. The user can add another turn to the same session; the timeline expands with the full conversation history.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ConversationEndpoint` | `HttpEndpoint` | `/api/conversations/*` — create session, add turn, list, get, SSE; serves `/api/metadata/*`. | — | `ConversationEntity`, `ConversationView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ConversationEntity` | `EventSourcedEntity` | Per-session lifecycle: audio received → transcript sanitized → responding → reply recorded → audio synthesised. Source of truth. | `ConversationEndpoint`, `TranscriptSanitizer`, `ConversationWorkflow` | `ConversationView` |
| `TranscriptSanitizer` | `Consumer` | Subscribes to `AudioReceived` events; redacts PII; calls `ConversationEntity.attachSanitizedTranscript`. | `ConversationEntity` events | `ConversationEntity` |
| `ConversationWorkflow` | `Workflow` | One workflow per turn. Steps: `awaitSanitizedStep` → `respondStep` → `synthesiseStep`. | started by `TranscriptSanitizer` once sanitized event lands | `VoiceConversationAgent`, `ConversationEntity` |
| `VoiceConversationAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the conversation history and persona in the task definition and the sanitized transcript as a task attachment; returns `AgentReply`. | invoked by `ConversationWorkflow` | returns reply |
| `ReplyGuardrail` | supporting class | `before-agent-response` hook on `VoiceConversationAgent`. Validates reply text length, disallowed phrases, and required JSON structure. | wired on agent | agent loop |
| `AudioSynthesiser` | supporting class | Converts reply text to audio bytes (stub: returns WAV header + silence; real providers wired in production). Runs inside `synthesiseStep`. | invoked by `ConversationWorkflow` | returns audio bytes |
| `ConversationView` | `View` | Read model: one row per session turn for the UI. | `ConversationEntity` events | `ConversationEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record TurnRequest(
    String conversationId,
    String turnId,
    String callerId,
    String rawTranscript,
    String audioFormat,        // e.g. "wav", "mp3", "simulated"
    Instant receivedAt
) {}

record SanitizedTranscript(
    String redactedTranscript,
    List<String> piiCategoriesFound
) {}

record AgentReply(
    String replyText,
    VoiceTone tone,
    String topicClassification,  // e.g. "billing", "technical", "cancellation"
    boolean escalationFlag,
    Instant repliedAt
) {}
enum VoiceTone { WARM, NEUTRAL, FIRM }

record AudioResponse(
    byte[] audioBytes,
    String audioFormat,
    int durationMs,
    Instant synthesisedAt
) {}

record Turn(
    String turnId,
    Optional<TurnRequest> request,
    Optional<SanitizedTranscript> sanitized,
    Optional<AgentReply> reply,
    Optional<AudioResponse> audio,
    TurnStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum TurnStatus {
    AUDIO_RECEIVED, TRANSCRIPT_SANITIZED, RESPONDING, REPLY_RECORDED, AUDIO_SYNTHESISED, FAILED
}

record ConversationSession(
    String conversationId,
    String callerId,
    List<Turn> turns,
    SessionStatus sessionStatus,
    Instant startedAt,
    Optional<Instant> closedAt
) {}

enum SessionStatus { ACTIVE, CLOSED, FAILED }
```

Events on `ConversationEntity`: `AudioReceived`, `TranscriptSanitized`, `ResponseStarted`, `ReplyRecorded`, `AudioSynthesised`, `TurnFailed`, `SessionClosed`.

Every nullable lifecycle field on `Turn` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/conversations` — body `{ callerId, rawTranscript, audioFormat }` → `{ conversationId, turnId }`.
- `POST /api/conversations/{id}/turns` — body `{ rawTranscript, audioFormat }` → `{ turnId }`.
- `GET /api/conversations` — list all active sessions, newest-first.
- `GET /api/conversations/{id}` — one session with all turns.
- `GET /api/conversations/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Voice Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of active sessions (status pill + tone badge + age + caller id) and a right pane with the selected session's turn timeline — each turn showing sanitized transcript, reply text, voice-tone badge, topic chip, escalation flag, and an audio playback button.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `TranscriptSanitizer` Consumer): redacts caller names, phone numbers, account numbers, payment-card-like tokens, email addresses, postal addresses, and date-of-birth patterns from the raw transcript before any LLM call. Records which categories were found.
- **G1 — before-agent-response guardrail**: runs on every turn of `VoiceConversationAgent`. Asserts the candidate response is well-formed `AgentReply` JSON, the `replyText` is within 500 characters (voice output budget), no entry in the disallowed-phrases list appears in the `replyText`, and the `tone` field is one of `{WARM, NEUTRAL, FIRM}`. On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.

## 9. Agent prompts

- `VoiceConversationAgent` → `prompts/voice-conversation.md`. The single decision-making LLM. System prompt instructs it to read the attached sanitized transcript, maintain an appropriate CX tone, stay within the response length budget, and return a typed `AgentReply`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a billing-query scenario transcript; within 30 s the reply appears with a non-empty `replyText`, a valid `tone`, a `topicClassification`, and an audio playback button.
2. **J2** — The agent's first response on a turn includes a disallowed phrase (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration produces a clean reply; the UI never displays the rejected response.
3. **J3** — A transcript containing `John Smith`, `0800-123-4567`, and `account ref 99887766` is submitted; the LLM call log shows only `[REDACTED-NAME]`, `[REDACTED-PHONE]`, `[REDACTED-ACCOUNT]`; the entity's `rawTranscript` retains the originals.
4. **J4** — A second turn is added to an existing session; the UI timeline shows both turns in correct order with their respective replies and audio buttons.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named voice-agent demonstrating the single-agent × cx-support cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-cx-support-voice-agent. Java package io.akka.samples.voiceagent. Akka 3.6.0.
HTTP port 9873.

Components to wire (exactly):

- 1 AutonomousAgent VoiceConversationAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/voice-conversation.md>) and
  .capability(TaskAcceptance.of(VOICE_RESPOND).maxIterationsPerTask(3)). The task receives
  the conversation history and persona as its instruction text and the sanitized transcript
  as a task ATTACHMENT (NOT as inline prompt text — Akka's TaskDef.attachment(name,
  contentBytes) is the canonical call). Output: AgentReply{replyText: String, tone:
  VoiceTone (WARM/NEUTRAL/FIRM), topicClassification: String, escalationFlag: boolean,
  repliedAt: Instant}. The agent is configured with a before-agent-response guardrail (see
  G1 in eval-matrix.yaml) registered via the agent's guardrail-configuration block. On
  guardrail rejection the agent loop retries within its 3-iteration budget.

- 1 Workflow ConversationWorkflow per turnId with three steps:
  * awaitSanitizedStep — polls ConversationEntity.getSession every 1s; on the current
    turn's sanitized().isPresent() advances to respondStep. WorkflowSettings.stepTimeout
    15s (sanitizer is in-process and fast).
  * respondStep — emits ResponseStarted, then calls componentClient.forAutonomousAgent(
    VoiceConversationAgent.class, "voice-" + turnId).runSingleTask(
      TaskDef.instructions(formatHistory(session.turns))
        .attachment("transcript.txt", turn.sanitized.redactedTranscript.getBytes())
    ) — returns a taskId, then forTask(taskId).result(VOICE_RESPOND) to fetch the reply.
    On success calls ConversationEntity.recordReply(reply). WorkflowSettings.stepTimeout
    60s with defaultStepRecovery maxRetries(2).failoverTo(ConversationWorkflow::error).
  * synthesiseStep — calls AudioSynthesiser.synthesise(reply.replyText) (stub: returns a
    minimal WAV; real providers swap in without changing this step). Calls
    ConversationEntity.recordAudio(audio). WorkflowSettings.stepTimeout 10s. error step
    transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ConversationEntity (one per conversationId). State ConversationSession
  {conversationId: String, callerId: String, turns: List<Turn>, sessionStatus: SessionStatus,
  startedAt: Instant, closedAt: Optional<Instant>}. Per-turn lifecycle embedded in Turn
  records; TurnStatus enum: AUDIO_RECEIVED, TRANSCRIPT_SANITIZED, RESPONDING, REPLY_RECORDED,
  AUDIO_SYNTHESISED, FAILED. SessionStatus: ACTIVE, CLOSED, FAILED. Events: AudioReceived
  {request}, TranscriptSanitized{turnId, sanitized}, ResponseStarted{turnId},
  ReplyRecorded{turnId, reply}, AudioSynthesised{turnId, audio}, TurnFailed{turnId, reason},
  SessionClosed{}. Commands: createSession, addTurn, attachSanitizedTranscript,
  markResponding, recordReply, recordAudio, failTurn, closeSession, getSession.
  emptyState() returns ConversationSession.initial("") with empty turns list and
  sessionStatus = ACTIVE (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside event-appliers.

- 1 Consumer TranscriptSanitizer subscribed to ConversationEntity events; on AudioReceived
  runs a regex+heuristic redaction pipeline (person names, phone numbers, account numbers,
  payment-card-like, email addresses, postal addresses, DOB patterns) over rawTranscript,
  computes the list of categories found, builds SanitizedTranscript, then calls
  ConversationEntity.attachSanitizedTranscript(turnId, sanitized). After
  attachSanitizedTranscript lands, the same Consumer starts a ConversationWorkflow with
  id = "turn-" + turnId.

- 1 View ConversationView with row type ConversationRow (mirrors ConversationSession minus
  per-turn rawTranscript — the audit log keeps the raw; the view holds sanitized forms for
  the UI). Table updater consumes ConversationEntity events. ONE query getAllSessions:
  SELECT * AS sessions FROM conversation_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ConversationEndpoint at /api with POST /conversations (body {callerId, rawTranscript,
    audioFormat}; mints conversationId + turnId; calls ConversationEntity.createSession +
    addTurn; returns {conversationId, turnId}), POST /conversations/{id}/turns (body
    {rawTranscript, audioFormat}; mints turnId; calls ConversationEntity.addTurn; returns
    {turnId}), GET /conversations (list from getAllSessions, sorted newest-first), GET
    /conversations/{id} (one session with all turns), GET /conversations/sse (Server-Sent
    Events forwarded from the view's stream-updates), and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ConversationTasks.java declaring one Task<R> constant: VOICE_RESPOND = Task.name("Respond
  to caller").description("Read the sanitized transcript attachment and produce an AgentReply
  for the caller").resultConformsTo(AgentReply.class). DO NOT skip this — the AutonomousAgent
  requires its companion Tasks class (Lesson 7).

- Domain records TurnRequest, SanitizedTranscript, AgentReply, VoiceTone, AudioResponse,
  Turn, TurnStatus, ConversationSession, SessionStatus.

- ReplyGuardrail.java implementing the before-agent-response hook. Reads the candidate
  AgentReply from the LLM response, runs the four checks listed in eval-matrix.yaml G1
  (well-formed JSON, replyText ≤ 500 chars, no disallowed phrase, tone in enum), and either
  passes the response through or returns Guardrail.reject(<structured-error>) to force the
  agent loop to retry.

- AudioSynthesiser.java — stub TTS engine (no external service). Inputs: reply text and
  target format. Outputs: AudioResponse. Production deployers swap in a real TTS provider
  by implementing the same interface. Documented in Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9873 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The VoiceConversationAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/scenarios.jsonl with 3 seeded conversation scenarios:
  a billing-query turn (caller upset about unexpected charge), a technical-support turn
  (caller cannot connect to the service), and a cancellation-request turn (caller wants to
  close their account). Each contains 2–3 plausible PII strings so S1 has work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (S1, G1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = autonomous-action
  (the agent's reply is spoken directly to the caller without a human intermediary),
  oversight.human_on_loop = true (supervisors can audit sessions; human_in_loop = false for
  normal turns), failure.failure_modes including "disallowed-output", "pii-leakage-via-llm",
  "off-script-escalation-bypass", "response-too-long-for-tts"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/voice-conversation.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Voice Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of session cards; right = selected-session turn timeline with sanitized
  transcripts, reply text, tone badges, topic chips, escalation flags, and audio buttons).
  Browser title exactly: <title>Akka Sample: Voice Agent</title>. No subtitle on the
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(turnId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    voice-respond.json — 8 AgentReply entries covering all three VoiceTone values and
      a spread of topicClassification values (billing, technical, cancellation). Each entry
      has a non-empty replyText (≤ 500 chars), a valid tone enum, a non-empty
      topicClassification, an escalationFlag, and a repliedAt timestamp. Plus 2 deliberately
      MALFORMED entries (one with replyText > 500 chars; one with a tone value outside the
      enum) — the guardrail blocks both, exercising the retry path. The mock should select
      a malformed entry on the FIRST iteration of every 3rd turn (modulo seed) so J2 is
      reproducible.
- A MockModelProvider.seedFor(turnId) helper makes per-turn selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. VoiceConversationAgent
  extends akka.javasdk.agent.autonomous.AutonomousAgent. The companion
  ConversationTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout
  (respondStep 60s, awaitSanitizedStep 15s, synthesiseStep 10s, error 5s).
- Lesson 6: every nullable lifecycle field on the Turn record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: ConversationTasks.java with VOICE_RESPOND = Task.name(...).description(...)
  .resultConformsTo(AgentReply.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9873 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words in narrative prose.
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (VoiceConversationAgent).
  AudioSynthesiser is a deterministic stub class — it does NOT make an LLM call.
- The transcript is passed as a Task ATTACHMENT, never inlined into the agent's instructions.
  Verify the generated respondStep uses TaskDef.attachment(...) and not string interpolation
  into the instruction text.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent returns.
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
