# SPEC — inbox-watcher

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Inbox Watcher.
**One-line pitch:** A background worker watches an inbox, redacts PII, drafts AI replies, and holds every draft for human approval before sending.

## 2. What this blueprint demonstrates

The **continuous-monitor** coordination pattern wired with four governance mechanisms layered on top of a single AI primitive (`DraftReplyAgent`). Specifically:

- A **PII sanitizer** runs inside a Consumer between the raw inbox event and the LLM call — so the model never sees identifiers.
- A **before-tool-call guardrail** on the (simulated) `sendReply` tool enforces "drafts must stay drafts until a human approves".
- A **HITL approve-to-send** gate is the actual production safeguard: the AI never sends; only an authenticated human can transition a drafted reply to SENT.
- An **eval-periodic** sampler running every 30 minutes scores a sample of sent replies for tone, accuracy, and policy adherence.

The result is a system where every reply has had four independent checks before it leaves the building.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live inbox: every received message, its sanitized payload, its classification, and (if drafted) the proposed reply.
2. `InboxPoller` (TimedAction) ticks every 15 s and inserts new simulated messages into `MessageQueue`. (A `RequestSimulator` style — drips canned messages.)
3. For each new message: `PiiSanitizer` (Consumer) redacts the payload, then `InboxFilterAgent` classifies it.
4. If the classification is `auto-drafted`, `DraftReplyAgent` produces a reply. The message lands in `AWAITING_APPROVAL`.
5. The human clicks Approve in the UI — the message transitions to SENT (simulated; no real outbound).
6. The human clicks Reject in the UI with a reason — the message transitions to DISMISSED.
7. `EvalRunner` (TimedAction) ticks every 30 minutes, picks N sent messages without `evalScore`, calls a judge agent, and writes a score back via an `EvalScored` event.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `InboxPoller` | `TimedAction` | Drips simulated messages into `MessageQueue` every 15 s. | scheduler | `MessageQueue` |
| `MessageQueue` | `EventSourcedEntity` | Append-only log of `MessageReceived` events. | `InboxPoller`, `InboxEndpoint` | `PiiSanitizer` |
| `PiiSanitizer` | `Consumer` | Reads `MessageReceived` events, applies regex+heuristic redaction, emits `MessageSanitized` via `InboxMessageEntity`. | `MessageQueue` events | `InboxMessageEntity` |
| `InboxFilterAgent` | `Agent` (typed, not autonomous) | Classifies sanitized message into `auto-drafted` / `escalate` / `ignore`. | invoked by Workflow | returns ClassificationResult |
| `DraftReplyAgent` | `AutonomousAgent` | Drafts a reply for `auto-drafted` messages. | invoked by Workflow | returns DraftedReply |
| `InboxWorkflow` | `Workflow` | Per-message orchestration: sanitize → classify → (maybe draft) → wait for human. | `PiiSanitizer` (one workflow per `MessageSanitized`) | `InboxMessageEntity` |
| `InboxMessageEntity` | `EventSourcedEntity` | Lifecycle per message: received → sanitized → classified → drafted → approved/rejected → sent/dismissed. | `InboxWorkflow` | `InboxView` |
| `InboxView` | `View` | Read-model row per message for the UI. | `InboxMessageEntity` events | `InboxEndpoint` |
| `EvalRunner` | `TimedAction` | Every 30 min, samples sent messages without scores; calls judge; writes `EvalScored`. | scheduler | `InboxMessageEntity` |
| `InboxEndpoint` | `HttpEndpoint` | `/api/inbox/*` — list, get, approve, reject, SSE. | — | `InboxView`, `InboxMessageEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record IncomingMessage(String messageId, String from, String subject, String body, Instant receivedAt) {}

record SanitizedPayload(String redactedSubject, String redactedBody, List<String> piiCategoriesFound) {}

record ClassificationResult(Classification kind, String confidence, String reason) {}
enum Classification { AUTO_DRAFTED, ESCALATE, IGNORE }

record DraftedReply(String subject, String body, Instant draftedAt) {}

record ApprovalDecision(boolean approved, String decidedBy, Optional<String> reason, Instant decidedAt) {}

record InboxMessage(
    String messageId,
    IncomingMessage incoming,
    Optional<SanitizedPayload> sanitized,
    Optional<ClassificationResult> classification,
    Optional<DraftedReply> draft,
    Optional<ApprovalDecision> decision,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    MessageStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum MessageStatus {
    RECEIVED, SANITIZED, CLASSIFIED, DRAFTED, AWAITING_APPROVAL, SENT, DISMISSED, ESCALATED
}
```

Events on `InboxMessageEntity`: `MessageReceived`, `MessageSanitized`, `MessageClassified`, `ReplyDrafted`, `ReplyApproved`, `ReplyRejected`, `MessageSent`, `MessageDismissed`, `MessageEscalated`, `EvalScored`.

Events on `MessageQueue`: `MessageReceived` (re-emitted into the queue as the audit log).

See `reference/data-model.md`.

## 6. API contract

- `GET /api/inbox` — list all messages. Optional `?status=…`.
- `GET /api/inbox/{id}` — one message.
- `POST /api/inbox/{id}/approve` — body `{ decidedBy }` → transitions DRAFTED to SENT.
- `POST /api/inbox/{id}/reject` — body `{ decidedBy, reason }` → transitions to DISMISSED.
- `GET /api/inbox/sse` — Server-Sent Events for every message change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Inbox Watcher</title>`.

App UI tab is the most distinctive: it shows the **live inbox** with a column per status, drag-to-approve interactions are out of scope — Approve/Reject are click-buttons inside each card.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied in `PiiSanitizer` Consumer): redacts emails, phone numbers, SSNs, addresses, account IDs from the message body before any LLM sees it. Records the categories found.
- **G1 — before-tool-call guardrail** on the `sendReply` simulated tool: blocks the call unless an `ApprovalDecision` with `approved=true` exists in entity state.
- **H1 — HITL approve-to-send** gate: the workflow explicitly pauses in AWAITING_APPROVAL; only `InboxEndpoint`'s approve/reject endpoints can advance it.
- **E1 — eval-periodic** (every 30 minutes): scores sent replies on tone/accuracy/policy.

## 9. Agent prompts

- `InboxFilterAgent` → `prompts/inbox-filter.md`. Typed classifier. Always returns one of the three `Classification` values.
- `DraftReplyAgent` → `prompts/draft-reply.md`. Drafts a reply staying within the constraints of `SanitizedPayload` (it never sees the raw message).
- `EvalJudge` (used by `EvalRunner`) → `prompts/eval-judge.md`. Scores a sent reply on a 1–5 rubric.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a message; it appears in the UI within 15 s; passes sanitize → classify → draft → AWAITING_APPROVAL.
2. **J2** — Click Approve; message transitions to SENT; the simulated outbound is logged but no network call leaves.
3. **J3** — Click Reject with reason; message transitions to DISMISSED with the reason visible.
4. **J4** — Eval Runner scores at least one SENT message within 30 minutes of system start; the score and rationale appear in the UI.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named inbox-watcher demonstrating the continuous-monitor × cx-support
cell. Runs out of the box (in-memory inbox source; no real email integration). Maven group
io.akka.samples. Java package io.akka.samples.inboxwatcher. HTTP port 9008.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) InboxFilterAgent — classifier. System prompt loaded from
  prompts/inbox-filter.md. Input: SanitizedPayload{redactedSubject, redactedBody,
  piiCategoriesFound: List<String>}. Output: ClassificationResult{kind: Classification (enum
  AUTO_DRAFTED/ESCALATE/IGNORE), confidence: "high"|"medium"|"low", reason: String}. Defaults
  to ESCALATE under uncertainty.

- 1 AutonomousAgent DraftReplyAgent — definition() with capability(TaskAcceptance.of(DRAFT)
  .maxIterationsPerTask(3)). System prompt from prompts/draft-reply.md. Input:
  SanitizedPayload + Optional<String> customerContext. Output: DraftedReply{subject, body,
  draftedAt}. The agent NEVER calls a send tool directly.

- 1 AutonomousAgent EvalJudge — definition() with capability(TaskAcceptance.of(EVAL)
  .maxIterationsPerTask(2)). System prompt from prompts/eval-judge.md. Input: SanitizedPayload
  + DraftedReply. Output: EvalResult{score: Integer 1–5, rationale: String}.

- 1 Workflow InboxWorkflow per message with steps: classifyStep -> conditional branch:
  AUTO_DRAFTED -> draftStep -> awaitApprovalStep -> finaliseStep; ESCALATE -> ends with
  MessageEscalated; IGNORE -> ends with MessageDismissed(reason="filter:IGNORE").
  classifyStep wraps the InboxFilterAgent call with WorkflowSettings.builder()
  .stepTimeout(Duration.ofSeconds(10)). draftStep wraps DraftReplyAgent with stepTimeout 30s.
  awaitApprovalStep polls InboxMessageEntity.getMessage every 5s; on decision.isPresent()
  advances. No auto-timeout on awaitApproval — drafts wait indefinitely. finaliseStep emits
  MessageSent or MessageDismissed based on decision.approved.

- 2 EventSourcedEntities:
  * MessageQueue — append-only audit log of inbound messages. Command receive(IncomingMessage)
    emits MessageReceived{incoming}.
  * InboxMessageEntity (one per messageId) — full per-message lifecycle. State
    InboxMessage{messageId, incoming: IncomingMessage{messageId, from, subject, body,
    receivedAt}, Optional<SanitizedPayload> sanitized, Optional<ClassificationResult>
    classification, Optional<DraftedReply> draft, Optional<ApprovalDecision> decision (with
    approved: boolean, decidedBy, Optional<String> reason, decidedAt), Optional<Integer>
    evalScore, Optional<String> evalRationale, MessageStatus status, Instant createdAt,
    Optional<Instant> finishedAt}. MessageStatus enum: RECEIVED, SANITIZED, CLASSIFIED,
    DRAFTED, AWAITING_APPROVAL, SENT, DISMISSED, ESCALATED. Events: MessageReceived,
    MessageSanitized, MessageClassified, ReplyDrafted, ReplyApproved, ReplyRejected,
    MessageSent, MessageDismissed, MessageEscalated, EvalScored. Commands: registerIncoming,
    attachSanitized, attachClassification, attachDraft, recordApproval, recordRejection,
    markSent, markDismissed, markEscalated, recordEval, getMessage. emptyState() returns
    InboxMessage.initial("", null) without commandContext() reference.

- 1 Consumer PiiSanitizer subscribed to MessageQueue events; for each MessageReceived,
  applies a regex+heuristic redaction pipeline (emails, phone numbers, SSNs, addresses,
  account-like tokens) to subject + body, builds SanitizedPayload with piiCategoriesFound,
  and calls InboxMessageEntity.registerIncoming followed by attachSanitized. Then starts
  an InboxWorkflow with messageId as the workflow id.

- 1 View InboxView with row type InboxMessageRow (mirrors InboxMessage minus raw incoming
  body). Table updater consumes InboxMessageEntity events. ONE query getAllMessages
  SELECT * AS messages FROM message_view. No WHERE status filter — caller filters client-side.

- 2 TimedActions:
  * InboxPoller — every 15s, reads next line from src/main/resources/sample-events/
    incoming-messages.jsonl and calls MessageQueue.receive.
  * EvalRunner — every 30 minutes, queries InboxView.getAllMessages, picks up to 5 SENT
    messages without an evalScore (oldest-first), calls EvalJudge with a 1–5 rubric per
    message, then calls InboxMessageEntity.recordEval(score, rationale) per message.

- 2 HttpEndpoints:
  * InboxEndpoint at /api with GET /inbox, GET /inbox/{id}, POST /inbox/{id}/approve (body
    {decidedBy}), POST /inbox/{id}/reject (body {decidedBy, reason}), GET /inbox/sse, and
    three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/. Approve writes ReplyApproved + MessageSent; reject writes
    ReplyRejected + MessageDismissed.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- InboxTasks.java declaring three Task<R> constants: CLASSIFY (ClassificationResult),
  DRAFT (DraftedReply), EVAL (EvalResult).
- Domain records IncomingMessage, SanitizedPayload, ClassificationResult, DraftedReply,
  ApprovalDecision, EvalResult.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9008 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars.
- src/main/resources/sample-events/incoming-messages.jsonl with 8 canned message lines
  covering AUTO_DRAFTED, ESCALATE, and IGNORE classifications.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 4 controls: S1 sanitizer pii, G1 guardrail
  before-tool-call scope-enforcement, H1 hitl application, E1 eval-periodic performance-monitor.
  Matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = draft-only,
  oversight.human_in_loop = true, oversight.reviewer_must_approve_every_outgoing = true,
  failure.failure_modes including "pii-leakage-via-llm" and "missed-escalation-of-urgent-
  message"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/inbox-filter.md, prompts/draft-reply.md, prompts/eval-judge.md loaded as agent
  system prompts.
- README.md at the project root: title "Akka Sample: Inbox Watcher", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar with the App UI tab using a two-column layout
  (left = live message list with classification chips and PII category chips; right = selected
  message detail with editable draft textarea + Approve/Reject buttons). Browser title
  exactly: <title>Akka Sample: Inbox Watcher</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the
        JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE (env-var name, file path, secrets URI); the value lives in the
  user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The message
  must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java implementing the ModelProvider interface
  with a per-agent dispatch on the agent class or Task<R> id. Each branch
  reads src/main/resources/mock-responses/<agent>.json, picks one entry
  pseudo-randomly per call, and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    inbox-filter.json — 8–12 ClassificationResult entries spanning
      AUTO_DRAFTED (routine support questions), ESCALATE (refunds, complaints,
      account-compromise patterns), and IGNORE (marketing, out-of-office,
      bounces). Each entry has confidence and a one-sentence reason. The
      mock should ESCALATE under ambiguity — match the prompt's default-
      to-escalation behaviour.
    draft-reply.json — 4–6 DraftedReply entries with subject prefixed "Re:",
      3–5 paragraph bodies, signed "— Customer Support Team". No invented
      policy timelines; no echoed [REDACTED] tokens.
    eval-judge.json — 6–8 EvalResult entries with score 1–5 and one-sentence
      rationales matching the rubric (tone / accuracy / policy adherence).
- A `MockModelProvider.seedFor(messageId)` helper makes per-message
  selection deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Optional<T> for every nullable field on a View row record.
- WorkflowSettings is nested inside Workflow — no import needed.
- emptyState() never calls commandContext().
- AutonomousAgent never silently downgraded to Agent.
- PiiSanitizer runs INSIDE a Consumer before any LLM call — not inside an Agent's prompt and
  not after the LLM has seen the raw payload.
- The agent NEVER sends; only the human Approve endpoint can transition to SENT.
- The generated static-resources/index.html must include the mermaid CSS
  overrides AND theme variables from Lesson 24 (state-diagram label colour,
  edge-label foreignObject overflow:visible, transitionLabelColor #cccccc).
  Without these, state names render black-on-black and arrow labels clip.
- Tab switching in static-resources/index.html MUST match by data-tab /
  data-panel attribute, NEVER by NodeList index (Lesson 26). No "hidden"
  zombie panels in the DOM — delete removed tabs, do not display:none them.
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var
  export block. Per Lesson 25, /akka:specify handles the key during
  generation.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK in
  narrative, T1/T2/T3/T4, marketing tone, competitor brand names.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.env` file written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
