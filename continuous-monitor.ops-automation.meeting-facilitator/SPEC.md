# SPEC — meeting-facilitator

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Teams Facilitator.
**One-line pitch:** A background worker monitors meeting sessions, sanitizes transcripts, produces structured notes and chat summaries with AI, and gates every publication behind a response guardrail before participants see it.

## 2. What this blueprint demonstrates

The **continuous-monitor** coordination pattern wired with two governance mechanisms layered over a dual-agent summary pipeline. Specifically:

- A **PII sanitizer** runs inside a Consumer between the raw transcript event and any LLM call — speaker names, email addresses, and phone numbers are redacted before the model processes a single word.
- A **before-agent-response guardrail** inspects every summary produced by the AI before it is written to the published state. Summaries that contain prohibited patterns (raw names, confidential markers, policy-violating content) are suppressed rather than published.

Together these ensure that a meeting summary distributed to all participants has been checked twice: once at ingestion (no PII in) and once at output (no policy violation out).

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live session list: every active or completed session, its sanitized transcript snippet, its generated notes, and (if published) the summary card.
2. `MeetingSessionPoller` (TimedAction) ticks every 20 s and inserts new simulated transcript segments into `MeetingSessionQueue`. (A simulator style — drips canned segments from a JSONL file.)
3. For each new segment batch: `TranscriptSanitizer` (Consumer) redacts the payload, then `MeetingFacilitatorWorkflow` orchestrates summarization.
4. `NotesSummaryAgent` produces structured meeting notes from the sanitized transcript. `ChatSummaryAgent` distils the chat thread.
5. The before-agent-response guardrail evaluates both outputs. If either fails, the session transitions to SUPPRESSED.
6. If both pass, the session transitions to PUBLISHED and the summary is available to participants.
7. `EvalRunner` (TimedAction) ticks every 30 minutes, picks N published sessions without `evalScore`, calls a judge agent, and writes a score back via an `EvalScored` event.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `MeetingSessionPoller` | `TimedAction` | Drips simulated transcript segments every 20 s. | scheduler | `MeetingSessionQueue` |
| `MeetingSessionQueue` | `EventSourcedEntity` | Append-only log of `SegmentReceived` events. | `MeetingSessionPoller`, `MeetingEndpoint` | `TranscriptSanitizer` |
| `TranscriptSanitizer` | `Consumer` | Reads `SegmentReceived` events, applies regex+heuristic redaction, emits `SegmentSanitized` via `MeetingSessionEntity`. | `MeetingSessionQueue` events | `MeetingSessionEntity` |
| `NotesSummaryAgent` | `Agent` (typed, not autonomous) | Produces structured meeting notes from a sanitized segment batch. | invoked by Workflow | returns `MeetingNotes` |
| `ChatSummaryAgent` | `Agent` (typed, not autonomous) | Distils the chat thread into a concise recap. | invoked by Workflow | returns `ChatRecap` |
| `MeetingFacilitatorWorkflow` | `Workflow` | Per-session orchestration: sanitize → summarize notes → summarize chat → guardrail check → publish or suppress. | `TranscriptSanitizer` (one workflow per `SegmentSanitized`) | `MeetingSessionEntity` |
| `MeetingSessionEntity` | `EventSourcedEntity` | Lifecycle per session: active → sanitized → summarized → published/suppressed. | `MeetingFacilitatorWorkflow` | `MeetingSessionView` |
| `MeetingSessionView` | `View` | Read-model row per session for the UI. | `MeetingSessionEntity` events | `MeetingEndpoint` |
| `EvalRunner` | `TimedAction` | Every 30 min, samples published sessions without scores; calls judge; writes `EvalScored`. | scheduler | `MeetingSessionEntity` |
| `MeetingEndpoint` | `HttpEndpoint` | `/api/sessions/*` — list, get, SSE. | — | `MeetingSessionView`, `MeetingSessionEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record TranscriptSegment(String segmentId, String sessionId, String speakerRaw,
    String textRaw, String chatThreadRaw, Instant capturedAt) {}

record SanitizedSegment(String redactedTranscript, String redactedChat,
    List<String> piiCategoriesFound) {}

record MeetingNotes(String title, List<String> keyPoints, List<String> actionItems,
    Instant generatedAt) {}

record ChatRecap(String summary, int messageCount, Instant generatedAt) {}

record GuardrailVerdict(boolean passed, String reason) {}

record EvalResult(int score, String rationale) {}

record MeetingSession(
    String sessionId,
    TranscriptSegment rawSegment,
    Optional<SanitizedSegment> sanitized,
    Optional<MeetingNotes> notes,
    Optional<ChatRecap> chatRecap,
    Optional<GuardrailVerdict> guardrailVerdict,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    SessionStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SessionStatus {
    ACTIVE, SANITIZED, SUMMARIZED, PUBLISHED, SUPPRESSED
}
```

Events on `MeetingSessionEntity`: `SegmentReceived`, `SegmentSanitized`, `NotesSummarized`, `ChatSummarized`, `GuardrailPassed`, `GuardrailFailed`, `SessionPublished`, `SessionSuppressed`, `EvalScored`.

Events on `MeetingSessionQueue`: `SegmentReceived` (re-emitted as the audit log).

See `reference/data-model.md`.

## 6. API contract

- `GET /api/sessions` — list all sessions. Optional `?status=…`.
- `GET /api/sessions/{id}` — one session.
- `GET /api/sessions/sse` — Server-Sent Events for every session state change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Teams Facilitator</title>`.

App UI tab is the most distinctive: it shows the **live session list** on the left and the selected session detail (sanitized transcript, meeting notes, chat recap, guardrail verdict, eval score) on the right.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied in `TranscriptSanitizer` Consumer): redacts speaker names, emails, phone numbers, and meeting-join links from the transcript and chat thread before any LLM call. Records the categories found.
- **G1 — before-agent-response guardrail** on every summary produced by `NotesSummaryAgent` and `ChatSummaryAgent`: blocks publication if the output contains raw PII patterns, confidential-meeting markers, or content that violates the operator's summary policy.

## 9. Agent prompts

- `NotesSummaryAgent` → `prompts/notes-summary.md`. Typed summariser. Input: sanitized transcript batch. Output: `MeetingNotes` with title, key points, and action items.
- `ChatSummaryAgent` → `prompts/chat-summary.md`. Typed summariser. Input: sanitized chat thread. Output: `ChatRecap` with summary and message count.
- `SummaryEvalJudge` (used by `EvalRunner`) → `prompts/summary-eval-judge.md`. Scores a published summary on a 1–5 rubric.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a segment; it appears in the UI within 20 s; passes sanitize → summarize → guardrail → PUBLISHED.
2. **J2** — A segment containing raw PII names triggers redaction; the LLM call payload contains only redacted tokens.
3. **J3** — A canned segment designed to trip the guardrail produces a SUPPRESSED session with `guardrailVerdict.passed = false`.
4. **J4** — Eval Runner scores at least one PUBLISHED session within 30 minutes of system start; the score and rationale appear in the UI.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named meeting-facilitator demonstrating the continuous-monitor × ops-automation
cell. Runs out of the box (in-memory meeting source; no real Teams integration). Maven group
io.akka.samples. Artifact id continuous-monitor-ops-automation-meeting-facilitator. Java package
io.akka.samples.teamsfacilitator. HTTP port 9316.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) NotesSummaryAgent — structured meeting notes. System prompt
  loaded from prompts/notes-summary.md. Input: SanitizedSegment{redactedTranscript,
  redactedChat, piiCategoriesFound: List<String>}. Output: MeetingNotes{title: String,
  keyPoints: List<String>, actionItems: List<String>, generatedAt: Instant}.

- 1 Agent (typed, NOT autonomous) ChatSummaryAgent — chat thread recap. System prompt from
  prompts/chat-summary.md. Input: SanitizedSegment. Output: ChatRecap{summary: String,
  messageCount: int, generatedAt: Instant}.

- 1 AutonomousAgent SummaryEvalJudge — definition() with capability(TaskAcceptance.of(EVAL)
  .maxIterationsPerTask(2)). System prompt from prompts/summary-eval-judge.md. Input:
  SanitizedSegment + MeetingNotes + ChatRecap. Output: EvalResult{score: Integer 1–5,
  rationale: String}.

- 1 Workflow MeetingFacilitatorWorkflow per session with steps:
  sanitizeStep (no-op; triggered after SegmentSanitized) -> notesStep (call NotesSummaryAgent,
  stepTimeout 30s) -> chatStep (call ChatSummaryAgent, stepTimeout 20s) ->
  guardrailStep (evaluate GuardrailVerdict) -> publishStep (emit SessionPublished or
  SessionSuppressed based on verdict.passed).
  notesStep and chatStep can run sequentially; guardrailStep reads both outputs.
  On timeout escalate to SUPPRESSED.

- 2 EventSourcedEntities:
  * MeetingSessionQueue — append-only audit log of inbound segments. Command
    receive(TranscriptSegment) emits SegmentReceived{segment}.
  * MeetingSessionEntity (one per sessionId) — full per-session lifecycle. State
    MeetingSession{sessionId, rawSegment: TranscriptSegment{segmentId, sessionId,
    speakerRaw, textRaw, chatThreadRaw, capturedAt}, Optional<SanitizedSegment> sanitized,
    Optional<MeetingNotes> notes, Optional<ChatRecap> chatRecap,
    Optional<GuardrailVerdict> guardrailVerdict (passed: boolean, reason: String),
    Optional<Integer> evalScore, Optional<String> evalRationale, SessionStatus status,
    Instant createdAt, Optional<Instant> finishedAt}. SessionStatus enum: ACTIVE,
    SANITIZED, SUMMARIZED, PUBLISHED, SUPPRESSED. Events: SegmentReceived, SegmentSanitized,
    NotesSummarized, ChatSummarized, GuardrailPassed, GuardrailFailed, SessionPublished,
    SessionSuppressed, EvalScored. Commands: registerSegment, attachSanitized, attachNotes,
    attachChatRecap, recordGuardrailPass, recordGuardrailFail, markPublished, markSuppressed,
    recordEval, getSession. emptyState() returns MeetingSession.initial("", null) without
    commandContext() reference.

- 1 Consumer TranscriptSanitizer subscribed to MeetingSessionQueue events; for each
  SegmentReceived, applies regex+heuristic redaction pipeline (speaker names pattern,
  emails, phone numbers, meeting-join URLs, account tokens) to textRaw + chatThreadRaw,
  builds SanitizedSegment with piiCategoriesFound, calls MeetingSessionEntity.registerSegment
  followed by attachSanitized. Then starts a MeetingFacilitatorWorkflow with sessionId as
  the workflow id.

- 1 View MeetingSessionView with row type MeetingSessionRow (mirrors MeetingSession minus
  raw segment payload). Table updater consumes MeetingSessionEntity events. ONE query
  getAllSessions SELECT * AS sessions FROM session_view. No WHERE status filter — caller
  filters client-side.

- 2 TimedActions:
  * MeetingSessionPoller — every 20s, reads next line from src/main/resources/sample-events/
    meeting-segments.jsonl and calls MeetingSessionQueue.receive.
  * EvalRunner — every 30 minutes, queries MeetingSessionView.getAllSessions, picks up to 5
    PUBLISHED sessions without an evalScore (oldest-first), calls SummaryEvalJudge with a
    1–5 rubric per session, then calls MeetingSessionEntity.recordEval(score, rationale) per
    session.

- 2 HttpEndpoints:
  * MeetingEndpoint at /api with GET /sessions, GET /sessions/{id}, GET /sessions/sse, and
    three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/. The SSE stream pushes one event per state transition.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- FacilitatorTasks.java declaring two Task<R> constants: NOTES (MeetingNotes), EVAL (EvalResult).
- Domain records TranscriptSegment, SanitizedSegment, MeetingNotes, ChatRecap,
  GuardrailVerdict, EvalResult.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9316 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars.
- src/main/resources/sample-events/meeting-segments.jsonl with 8 canned segment lines
  covering routine meetings (PUBLISHED path), segments with PII names (sanitizer fires),
  and one deliberately policy-tripping segment (SUPPRESSED path).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: S1 sanitizer pii, G1 guardrail
  before-agent-response policy-enforcement. Matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = auto-publish,
  oversight.human_in_loop = false, oversight.human_on_loop = true,
  failure.failure_modes including "pii-in-published-summary" and
  "confidential-detail-exposed-to-wrong-audience"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/notes-summary.md, prompts/chat-summary.md, prompts/summary-eval-judge.md loaded
  as agent system prompts.
- README.md at the project root (see above).
- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar with the App UI tab using a two-column layout
  (left = live session list with status pills and PII category chips; right = selected
  session detail with meeting notes, chat recap, guardrail verdict badge, and eval score).
  Browser title exactly: <title>Akka Sample: Teams Facilitator</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java per-agent dispatch on agent class or Task<R> id.
  Each branch reads src/main/resources/mock-responses/<agent>.json, picks one
  entry pseudo-randomly per call, and deserialises into the agent's typed return.
- Per-agent mock-response shapes:
    notes-summary.json — 6–8 MeetingNotes entries with varied titles (sprint
      review, design sync, incident retrospective), 3–6 keyPoints each, 1–3
      actionItems each. No invented personal names.
    chat-summary.json — 4–6 ChatRecap entries with 1–3 sentence summaries,
      messageCount ranging 8–40.
    summary-eval-judge.json — 6–8 EvalResult entries with score 1–5 and
      one-sentence rationales covering tone, completeness, PII-cleanliness.
- A MockModelProvider.seedFor(sessionId) helper makes selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Optional<T> for every nullable field on a View row record.
- WorkflowSettings nested inside Workflow.
- emptyState() never calls commandContext().
- NotesSummaryAgent and ChatSummaryAgent are typed Agents — not AutonomousAgents.
- TranscriptSanitizer runs INSIDE a Consumer before any LLM call.
- The guardrail is a before-agent-response check: it inspects the agent output BEFORE
  it is persisted as the published summary. If it fails, SessionSuppressed is emitted
  instead of SessionPublished.
- The generated static-resources/index.html must include the mermaid CSS overrides AND
  theme variables from Lesson 24 (state-diagram label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc).
- Tab switching MUST match by data-tab / data-panel attribute, NEVER by NodeList index
  (Lesson 26). No hidden zombie panels in the DOM.
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var export block.
- No forbidden words in user-facing text.
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
