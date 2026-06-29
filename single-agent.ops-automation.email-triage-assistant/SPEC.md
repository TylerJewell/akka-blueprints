# SPEC — email-triage-assistant

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Multi-Modal Email Assistant.
**One-line pitch:** A user submits an email (body + attachments) to the triage service; one AI agent reads the sanitized message, classifies it by urgency and category, drafts a reply, and presents the draft for human confirmation before any outbound send is attempted.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the ops-automation domain. One `EmailTriageAgent` (AutonomousAgent) carries both the classification and the draft-generation decisions; the surrounding components only prepare its input, gate its tool calls, and audit its output. Three governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw email submission and the agent call — so the model never sees personal identifiers from the original message.
- A **before-tool-call guardrail** intercepts every invocation of the agent's `send_email` tool. Until `ConfirmationEntity` records an explicit user approval for this email's draft, the guardrail returns a structured block error and the send does not proceed. Once approved, the guardrail passes and the tool executes.
- A **human-in-the-loop** confirmation step is wired as a `TriageWorkflow.confirmStep` that pauses the workflow and surfaces the draft to the UI. The user clicks **Approve** or **Reject**; the workflow resumes on the entity event.

The blueprint shows that a single-agent ops workflow can be governed without a second LLM call: the guardrail and HITL check are structural, not model-powered.

## 3. User-facing flows

The user opens the App UI tab.

1. The user picks one of three seeded emails from a dropdown (a billing dispute, a product-support request, a contract-review enquiry) or pastes a raw email body.
2. The user optionally attaches a file by pasting its text content into the attachment area (supports plain-text extracted content; actual file upload is simulated in-process).
3. The user clicks **Submit for triage**. The UI POSTs to `/api/emails` and receives an `emailId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `SANITIZED` — the redacted body is visible in the detail pane, with a list of PII categories the sanitizer found.
5. Within ~10–30 s, the agent finishes. The card transitions to `REVIEWING` then `DRAFT_READY`. The triage panel shows: urgency badge (HIGH / MEDIUM / LOW), category chip, a short classification rationale, and the full draft reply text.
6. The user reads the draft and clicks **Approve** or **Reject** in the right pane. On approval the card transitions to `SENDING` and then `SENT`. On rejection it transitions to `REJECTED`.
7. The user can submit another email; the live list keeps history.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TriageEndpoint` | `HttpEndpoint` | `/api/emails/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `EmailEntity`, `TriageView`, `ConfirmationEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `EmailEntity` | `EventSourcedEntity` | Per-email lifecycle: submitted → sanitized → reviewing → draft-ready → sent/rejected. Source of truth. | `TriageEndpoint`, `EmailSanitizer`, `TriageWorkflow` | `TriageView` |
| `ConfirmationEntity` | `EventSourcedEntity` | Records the user's approve/reject decision per email. Notifies `TriageWorkflow`. | `TriageEndpoint` | `TriageWorkflow` |
| `EmailSanitizer` | `Consumer` | Subscribes to `EmailSubmitted` events; strips PII; calls `EmailEntity.attachSanitized`. | `EmailEntity` events | `EmailEntity` |
| `TriageWorkflow` | `Workflow` | One workflow per email. Steps: `awaitSanitizedStep` → `triageStep` → `confirmStep` → `sendStep`. | started by `EmailSanitizer` once sanitized event lands | `EmailTriageAgent`, `EmailEntity`, `ConfirmationEntity` |
| `EmailTriageAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the sanitized email body (and attachment text) as a task attachment; returns `TriageResult` with classification and draft. Has a `send_email` tool; all invocations are intercepted by `SendGuardrail`. | invoked by `TriageWorkflow` | returns `TriageResult` |
| `SendGuardrail` | supporting class | Implements `before-tool-call` hook on `EmailTriageAgent`. Checks `ConfirmationEntity` for an approval event before permitting the `send_email` tool call. | `EmailTriageAgent` tool calls | `ConfirmationEntity` |
| `TriageView` | `View` | Read model: one row per email for the UI. | `EmailEntity` events | `TriageEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record AttachmentContent(
    String filename,
    String textContent    // plain-text extracted content of the attachment
) {}

record EmailSubmission(
    String emailId,
    String fromAddress,
    String subject,
    String rawBody,
    List<AttachmentContent> attachments,
    String submittedBy,
    Instant submittedAt
) {}

record SanitizedEmail(
    String redactedBody,
    List<String> redactedAttachmentTexts,
    List<String> piiCategoriesFound
) {}

record TriageResult(
    Urgency urgency,
    Category category,
    String classificationRationale,
    String draftReply,
    Instant classifiedAt
) {}
enum Urgency { HIGH, MEDIUM, LOW }
enum Category { BILLING, SUPPORT, LEGAL, GENERAL }

record ConfirmationDecision(
    String emailId,
    ConfirmationStatus decision,
    String decidedBy,
    Instant decidedAt
) {}
enum ConfirmationStatus { APPROVED, REJECTED }

record Email(
    String emailId,
    Optional<EmailSubmission> submission,
    Optional<SanitizedEmail> sanitized,
    Optional<TriageResult> triage,
    Optional<ConfirmationDecision> confirmation,
    EmailStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum EmailStatus {
    SUBMITTED, SANITIZED, REVIEWING, DRAFT_READY, SENDING, SENT, REJECTED, FAILED
}
```

Events on `EmailEntity`: `EmailSubmitted`, `EmailSanitized`, `TriageStarted`, `DraftReady`, `ConfirmationReceived`, `EmailSent`, `EmailRejected`, `EmailFailed`.

Every nullable lifecycle field on the `Email` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/emails` — body `{ fromAddress, subject, rawBody, attachments: [AttachmentContent], submittedBy }` → `{ emailId }`.
- `GET /api/emails` — list all emails, newest-first.
- `GET /api/emails/{id}` — one email.
- `GET /api/emails/sse` — Server-Sent Events; one event per state transition.
- `POST /api/emails/{id}/confirm` — body `{ decision: "APPROVED" | "REJECTED", decidedBy }` → `204`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Multi-Modal Email Assistant</title>`.

The App UI tab is a two-column layout: a left column with a submission form and the live list of submitted emails (status pill + urgency badge + age) and a right pane with the selected email's detail — sanitized body preview, PII category chips, triage classification, draft reply text, and approve/reject buttons.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `EmailSanitizer` Consumer): strips emails, phone numbers, government identifiers, payment-card-like tokens, person names, postal addresses, and account-like identifiers from the raw email body and all attachment text content before any LLM call. Records which categories were found.
- **G1 — before-tool-call guardrail**: runs on every tool invocation inside `EmailTriageAgent`. Specifically targets the `send_email` tool. Before allowing execution, checks `ConfirmationEntity` for an `ConfirmationReceived(APPROVED)` event on the same `emailId`. If no approval exists yet, returns a structured block error to the agent loop indicating the send is pending human confirmation. Once an approval lands, the guardrail passes and the tool call executes.
- **H1 — human-in-the-loop confirmation** (`hitl`, `application`): implemented as `TriageWorkflow.confirmStep`. After `DraftReady` lands, the workflow pauses and waits for `ConfirmationEntity` to emit `ConfirmationReceived`. The UI's **Approve** / **Reject** buttons POST to `/api/emails/{id}/confirm`, which writes to `ConfirmationEntity`. The workflow polls `ConfirmationEntity` every 2 s up to its 300 s timeout, then advances to `sendStep` (on approval) or records `EmailRejected` (on rejection).

## 9. Agent prompts

- `EmailTriageAgent` → `prompts/email-triage-agent.md`. The single decision-making LLM. System prompt instructs it to read the sanitized email and attachments, classify urgency and category, and draft a reply. It has one tool: `send_email(to, subject, body)` — but calling it requires human approval via the `before-tool-call` guardrail.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the billing-dispute seed email; within 30 s the draft appears with `BILLING` category and `HIGH` urgency, and the approve/reject buttons appear.
2. **J2** — The user approves the draft; the `before-tool-call` guardrail confirms the approval in `ConfirmationEntity` and permits the `send_email` tool call; the entity transitions to `SENT`.
3. **J3** — The user rejects a draft; no outbound tool call is ever attempted; the entity transitions to `REJECTED`.
4. **J4** — A submitted email containing `jane.doe@example.com` and `+1 650 555 0100` is processed; the LLM call log shows only `[REDACTED-EMAIL]` and `[REDACTED-PHONE]`; the entity's `submission.rawBody` retains the raw text for audit.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named multi-modal-email-assistant demonstrating the single-agent ×
ops-automation cell. Runs out of the box (no external email service). Maven group
io.akka.samples. Maven artifact single-agent-ops-automation-email-triage-assistant.
Java package io.akka.samples.multimodalemailassistant. Akka 3.6.0. HTTP port 9664.

Components to wire (exactly):

- 1 AutonomousAgent EmailTriageAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/email-triage-agent.md>) and
  .capability(TaskAcceptance.of(TRIAGE_EMAIL).maxIterationsPerTask(3)). The agent
  has one declared tool: send_email(to: String, subject: String, body: String) that
  simulates outbound delivery. A before-tool-call guardrail (see G1 in eval-matrix.yaml)
  is registered via the agent's guardrail-configuration block; it intercepts every
  send_email invocation. Output: TriageResult{urgency: Urgency, category: Category,
  classificationRationale: String, draftReply: String, classifiedAt: Instant}. The
  email body and attachment texts are passed as task attachments (NOT inline prompt
  text — Akka's TaskDef.attachment(name, contentBytes) is the canonical call).

- 1 Workflow TriageWorkflow per emailId with four steps:
  * awaitSanitizedStep — polls EmailEntity.getEmail every 1s; on
    email.sanitized().isPresent() advances to triageStep.
    WorkflowSettings.stepTimeout 15s.
  * triageStep — emits TriageStarted, then calls
    componentClient.forAutonomousAgent(EmailTriageAgent.class, "triage-" + emailId)
    .runSingleTask(
      TaskDef.instructions(formatEmailContext(email.submission()))
        .attachment("body.txt", email.sanitized().redactedBody().getBytes())
        .attachment("attachments.txt", joinAttachmentTexts(email.sanitized()).getBytes())
    ) — fetches the result via forTask(taskId).result(TRIAGE_EMAIL).
    On success calls EmailEntity.recordDraft(triage). WorkflowSettings.stepTimeout
    90s with defaultStepRecovery maxRetries(2).failoverTo(TriageWorkflow::error).
  * confirmStep — polls ConfirmationEntity.getDecision every 2s up to 300s.
    On APPROVED advances to sendStep; on REJECTED calls EmailEntity.recordRejected.
    WorkflowSettings.stepTimeout 300s.
  * sendStep — invokes the send_email tool call path (simulated in mock mode) and
    calls EmailEntity.recordSent. WorkflowSettings.stepTimeout 15s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a
  top-level WorkflowSettings class (Lesson 5).

- 2 EventSourcedEntities:
  * EmailEntity (one per emailId). State Email{emailId: String,
    submission: Optional<EmailSubmission>, sanitized: Optional<SanitizedEmail>,
    triage: Optional<TriageResult>, confirmation: Optional<ConfirmationDecision>,
    status: EmailStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
    EmailStatus enum: SUBMITTED, SANITIZED, REVIEWING, DRAFT_READY, SENDING, SENT,
    REJECTED, FAILED. Events: EmailSubmitted{submission}, EmailSanitized{sanitized},
    TriageStarted{}, DraftReady{triage}, ConfirmationReceived{confirmation},
    EmailSent{sentAt}, EmailRejected{reason}, EmailFailed{reason}. Commands: submit,
    attachSanitized, markReviewing, recordDraft, recordConfirmation, recordSent,
    recordRejected, fail, getEmail. emptyState() returns Email.initial("") with no
    commandContext() reference (Lesson 3). Every Optional<T> field uses
    Optional.empty() in initial state and Optional.of(...) in the event-applier.
  * ConfirmationEntity (one per emailId). State ConfirmationState{emailId: String,
    decision: Optional<ConfirmationDecision>}. Events: DecisionRecorded{decision}.
    Commands: record, getDecision.

- 1 Consumer EmailSanitizer subscribed to EmailEntity events; on EmailSubmitted
  runs a regex+heuristic redaction pipeline (emails, phone numbers, SSN-like,
  payment-card-like, postal addresses, person-name heuristic, account-id-like tokens)
  over rawBody AND each attachment's textContent, computes the list of categories
  found, builds SanitizedEmail, then calls EmailEntity.attachSanitized(sanitized).
  After attachSanitized lands, the same Consumer starts a TriageWorkflow with
  id = "triage-" + emailId.

- 1 View TriageView with row type EmailRow (mirrors Email minus
  submission.rawBody — the audit log keeps the raw; the view holds the sanitized
  form for the UI). Table updater consumes EmailEntity events. ONE query
  getAllEmails: SELECT * AS emails FROM triage_view. No WHERE status filter — Akka
  cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * TriageEndpoint at /api with POST /emails (body {fromAddress, subject, rawBody,
    attachments: [{filename, textContent}], submittedBy}; mints emailId; calls
    EmailEntity.submit; returns {emailId}), GET /emails (list from getAllEmails,
    sorted newest-first), GET /emails/{id} (one row), GET /emails/sse (Server-Sent
    Events forwarded from the view's stream-updates), POST /emails/{id}/confirm
    (body {decision: "APPROVED"|"REJECTED", decidedBy}; calls
    ConfirmationEntity.record; returns 204), and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* ->
    static-resources/*.

Companion files:

- EmailTasks.java declaring one Task<R> constant: TRIAGE_EMAIL = Task.name("Triage
  email").description("Read the sanitized email and attachments, classify urgency and
  category, draft a reply, and optionally call send_email when the user approves")
  .resultConformsTo(TriageResult.class). DO NOT skip this — the AutonomousAgent
  requires its companion Tasks class (Lesson 7).

- Domain records AttachmentContent, EmailSubmission, SanitizedEmail, TriageResult,
  Urgency, Category, ConfirmationDecision, ConfirmationStatus, Email, EmailStatus.

- SendGuardrail.java implementing the before-tool-call hook. On each send_email
  invocation: reads ConfirmationEntity.getDecision for the current emailId; if
  decision is APPROVED passes the tool call through; if absent or REJECTED returns
  Guardrail.block("send-pending-confirmation") so the agent loop does not execute
  the send and surfaces the pending state.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9664
  and the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/seed-emails.jsonl with 3 seeded emails: a
  billing-dispute email (contains email address, phone number, account number), a
  product-support request (contains name, device serial number), and a
  contract-review enquiry (contains company name, signatory name). Each contains
  2–3 PII strings so S1 has work to do.

- src/main/resources/sample-events/seed-attachments.jsonl with 3 matching
  attachment texts: a PDF-extracted billing statement (500 words, contains PII), a
  screenshot OCR transcript (200 words), and a contract excerpt (800 words).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (S1, G1, H1) matching the
  mechanisms in Section 8. Matching simplified_view list.

- risk-survey.yaml at the project root pre-filled for the ops-automation domain.

- prompts/email-triage-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Multi-Modal Email Assistant",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms
  section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no
  ui/, no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column
  layout (left = submission form + live list; right = selected-email detail with
  sanitized body preview, PII chips, triage classification, draft reply, and
  approve/reject buttons). Browser title exactly:
  <title>Akka Sample: Multi-Modal Email Assistant</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-
        but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf;
        /akka:build forwards the value from the Claude session env.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured
  key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java with a per-task dispatch on TRIAGE_EMAIL. Each
  branch reads src/main/resources/mock-responses/triage-email.json, picks one
  entry pseudo-randomly per call (seedFor(emailId)), and deserialises into
  TriageResult.
- triage-email.json — 8 TriageResult entries covering the three Urgency and four
  Category values, each with a realistic classificationRationale and draftReply.
  Include 2 entries with deliberate issues (one where draftReply is empty, one where
  category is null) to exercise edge handling in the UI.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. EmailTriageAgent
  extends akka.javasdk.agent.autonomous.AutonomousAgent. The companion
  EmailTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout
  (triageStep 90s, awaitSanitizedStep 15s, confirmStep 300s, sendStep 15s,
  error 5s).
- Lesson 6: every nullable lifecycle field on the Email row record is Optional<T>.
- Lesson 7: EmailTasks.java with TRIAGE_EMAIL is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9664 declared explicitly in application.conf.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words.
- Lesson 24: index.html includes mermaid CSS overrides and themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by
  NodeList index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (EmailTriageAgent).
  The HITL confirmation and guardrail checks are structural — no second LLM call.
- Email body and attachments are passed as task ATTACHMENTS, never inlined into the
  agent's instructions. Verify the generated triageStep uses TaskDef.attachment(...)
  and not string interpolation into the instruction text.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration
  mechanism targeting the send_email tool specifically. It reads ConfirmationEntity
  state synchronously before deciding to pass or block.
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
