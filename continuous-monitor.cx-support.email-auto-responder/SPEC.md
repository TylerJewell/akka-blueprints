# SPEC — email-auto-responder

The natural-language brief `/akka:specify` reads to generate this system. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Email Auto Responder.
**One-line pitch:** The system watches a support inbox; for each inbound email it redacts PII, drafts a contextual reply with an agent, sends low-risk replies automatically, and pauses sensitive ones for a human to approve in the UI.

## 2. What this blueprint demonstrates

The continuous-monitor coordination pattern: a TimedAction continuously samples an inbound mail source and drips messages through an event-sourced queue into a Consumer that starts one workflow per email. The governance pattern wires three controls — a PII sanitizer on email content before it reaches the model, a before-tool-call guardrail on the outbound send action, and an application-level human-in-the-loop gate that holds sensitive replies until a person approves them.

## 3. User-facing flows

1. An inbound email arrives (the monitor drips one from the canned source, or a client POSTs one). The system ingests it and the UI shows it in `RECEIVED`.
2. The workflow sanitizes the body — phone numbers, emails, and card-like numbers are masked — and the UI shows the redacted body in `SANITIZED`.
3. The responder agent drafts a reply; triage marks it sensitive or routine. The UI shows the draft in `DRAFTED`.
4. A routine reply auto-sends; the UI shows it in `SENT` with a message id. A sensitive reply pauses in `AWAITING_REVIEW` with Approve/Reject buttons.
5. A human clicks Approve (reply sends, `SENT`) or Reject with a reason (`REJECTED`).
6. An email left in `AWAITING_REVIEW` past the threshold is escalated and the buttons disappear.

These become the acceptance journeys in `reference/user-journeys.md`.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `InboxMonitor` | TimedAction | Every 30s reads the next line from the canned inbound source and enqueues it | — | `InboxQueue` |
| `InboxQueue` | EventSourcedEntity | Single instance; `receive(email)` emits `EmailQueued` | `InboxMonitor`, `ResponderEndpoint` | `InboundEmailConsumer` |
| `InboundEmailConsumer` | Consumer | Subscribes to `InboxQueue` events; starts one `ResponderWorkflow` per email | `InboxQueue` | `ResponderWorkflow` |
| `ResponderWorkflow` | Workflow | Steps `sanitizeStep` → `draftStep` → `reviewGateStep` → `sendStep` | `InboundEmailConsumer` | `EmailEntity`, `TriageAgent`, `ResponderAgent` |
| `TriageAgent` | Agent | Classifies intent and sensitivity of a sanitized email | `ResponderWorkflow` | `ResponderWorkflow` |
| `ResponderAgent` | AutonomousAgent | Drafts a contextual reply; returns typed `DraftReply` | `ResponderWorkflow` | `ResponderWorkflow` |
| `EmailEntity` | EventSourcedEntity | Durable per-email state and lifecycle events | `ResponderWorkflow`, `ResponderEndpoint` | `EmailsView` |
| `EmailsView` | View | Row type `Email`; one query `getAllEmails` | `EmailEntity` | `ResponderEndpoint` |
| `StuckReviewMonitor` | TimedAction | Every 30s escalates emails in `AWAITING_REVIEW` older than the threshold | `EmailsView` | `EmailEntity` |
| `ResponderEndpoint` | HttpEndpoint | `/api/*` REST + SSE + metadata | UI, clients | `EmailEntity`, `EmailsView`, `InboxQueue` |
| `AppEndpoint` | HttpEndpoint | Serves `/` (redirect) and `/app/*` static UI | browser | static-resources |
| `Bootstrap` | service-setup | Schedules the two TimedActions on startup | — | `InboxMonitor`, `StuckReviewMonitor` |

Names are used verbatim by `/akka:specify`.

## 5. Data model

See `reference/data-model.md`. `Email` is the event-sourced state and the View row. Non-lifecycle fields (`id`, `fromAddress`, `subject`, `body`, `status`) are always present; every nullable lifecycle field is `Optional<T>` (Lesson 6): `receivedAt`, `sanitizedBody`, `draftSubject`, `draftBody`, `sensitive` (`Optional<Boolean>`), `draftedAt`, `approvedAt`, `approvedBy`, `rejectedAt`, `rejectedBy`, `rejectReason`, `sentAt`, `sentMessageId`, `escalatedAt`.

`EmailStatus` enum: `RECEIVED · SANITIZED · DRAFTED · AWAITING_REVIEW · APPROVED · REJECTED · SENT · ESCALATED`.

`EmailEvent` (sealed, 6 variants): `EmailIngested`, `EmailSanitized`, `ReplyDrafted`, `ReplyApproved`, `ReplyRejected`, `ReplySent`, `EmailEscalated`. Auxiliary records: `DraftReply(String subject, String body)`, `Triage(boolean sensitive, String reason)`.

## 6. API contract

Inline surface below; payload schemas and SSE event format in `reference/api-contract.md`.

```
POST /api/emails                       -> { emailId }       (ingest an inbound email)
POST /api/emails/{id}/approve          -> 200 | 404
POST /api/emails/{id}/reject           -> 200 | 404
GET  /api/emails ?status=...           -> { emails: [Email, ...] }   (filter client-side)
GET  /api/emails/{id}                  -> Email
GET  /api/emails/sse                   -> Server-Sent Events of Email
GET  /api/metadata/eval-matrix         -> text/yaml
GET  /api/metadata/risk-survey         -> text/yaml
GET  /api/metadata/readme              -> text/markdown
GET  /                                 -> 302 /app/index.html
GET  /app/{*path}                      -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` — no `ui/` folder, no npm build. Five tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI. Browser title `<title>Akka Sample: Email Auto Responder</title>`. Tab switching matches by `data-tab` / `data-panel` attribute, never NodeList index (Lesson 26). Mermaid diagrams on the Architecture tab carry the Lesson 24 CSS overrides for state-label colour and edge-label overflow. The App UI tab lists emails live via SSE and shows Approve/Reject on emails in `AWAITING_REVIEW`. See `reference/ui-mockup.md`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. Three controls: `S1` sanitizer redacts PII from the email body in `sanitizeStep` before it reaches the model or logs; `G1` before-tool-call guardrail blocks the simulated send tool unless the email status is `APPROVED` or the reply was triaged routine; `H1` application HITL pauses `reviewGateStep` for sensitive replies until a human approves or rejects.

## 9. Agent prompts

- `TriageAgent` → `prompts/triage-agent.md` — classifies a sanitized email's intent and whether the drafted reply needs human approval.
- `ResponderAgent` → `prompts/responder-agent.md` — drafts a concise, contextual support reply as a typed `DraftReply`.

## 10. Acceptance

Inline below; full steps in `reference/user-journeys.md`.

1. **Ingest and draft** — an inbound email reaches `DRAFTED` with PII masked in `sanitizedBody` and a non-empty `draftBody` within ~30s.
2. **Sensitive gate** — an email triaged sensitive pauses in `AWAITING_REVIEW`; Approve drives it to `SENT` with a non-null `sentMessageId`.
3. **Routine auto-send** — an email triaged routine reaches `SENT` with no human action.
4. **Escalation** — an `AWAITING_REVIEW` email left untouched past the threshold transitions to `ESCALATED`.

---

## 11. Implementation directives

```
Create a sample named email-auto-responder demonstrating the
continuous-monitor x cx-support cell. Runs out of the box (no external
services). Maven group io.akka.samples. Maven artifact email-auto-responder.
Java package io.akka.samples.emailautoresponder. Akka 3.6.0. HTTP port 9165.

Components to wire (exactly):
- 1 AutonomousAgent ResponderAgent: drafts a support reply, returns a typed
  DraftReply{subject,body}. definition() declares
  capability(TaskAcceptance.of(ResponderTasks.DRAFT_REPLY).maxIterationsPerTask(3)).
  Companion ResponderTasks.java declares Task<DraftReply> DRAFT_REPLY with
  Task.name(...).description(...).resultConformsTo(DraftReply.class).
- 1 Agent TriageAgent: classifies a sanitized email, returns Triage{sensitive,
  reason} via a method classify(String sanitizedBody) using
  effects().systemMessage(...).userMessage(...).responseAs(Triage.class).thenReply().
- 1 Workflow ResponderWorkflow with steps sanitizeStep -> draftStep ->
  reviewGateStep -> sendStep. sanitizeStep masks PII (regex for emails, phone
  numbers, 13-19 digit card-like runs) and calls EmailEntity.recordSanitized.
  draftStep calls forAgent(TriageAgent).classify then
  forAutonomousAgent(ResponderAgent.class, "resp-"+id).runSingleTask(...) and
  forTask(taskId).result(DRAFT_REPLY); calls EmailEntity.recordDraft.
  reviewGateStep: if Triage.sensitive, transition to AWAITING_REVIEW, self-
  schedule a 5-second resume timer, poll EmailEntity until APPROVED (->sendStep),
  REJECTED or ESCALATED (->end); if not sensitive, go straight to sendStep.
  sendStep: before-tool-call guardrail verifies status APPROVED or non-sensitive,
  simulates the send, calls EmailEntity.recordSent. Override settings() with
  stepTimeout 60s on draftStep and sendStep, 10s on sanitizeStep and
  reviewGateStep. WorkflowSettings is nested in Workflow (no import).
- 1 EventSourcedEntity EmailEntity holding an Email record: id, fromAddress,
  subject, body, EmailStatus status, and Optional lifecycle fields receivedAt,
  sanitizedBody, draftSubject, draftBody, sensitive (Optional<Boolean>),
  draftedAt, approvedAt, approvedBy, rejectedAt, rejectedBy, rejectReason,
  sentAt, sentMessageId, escalatedAt. Events EmailIngested, EmailSanitized,
  ReplyDrafted, ReplyApproved, ReplyRejected, ReplySent, EmailEscalated.
  Commands ingest, recordSanitized, recordDraft, approve, reject, recordSent,
  markEscalated, getEmail. emptyState() returns Email.initial("") with no
  commandContext() reference.
- 1 EventSourcedEntity InboxQueue, single instance "default", command
  receive(EmailIn) emitting EmailQueued.
- 1 View EmailsView, row type Email, table updater consuming EmailEntity events.
  ONE query getAllEmails: SELECT * AS emails FROM emails_view. No WHERE status
  filter (Akka cannot auto-index enum columns); filter client-side in callers.
- 1 Consumer InboundEmailConsumer subscribed to InboxQueue events; on each
  EmailQueued starts a ResponderWorkflow with a fresh UUID, seeding the
  EmailEntity via ingest first.
- 2 TimedActions: InboxMonitor (every 30s, reads the next line from
  src/main/resources/sample-events/inbound-emails.jsonl, calls
  InboxQueue.receive); StuckReviewMonitor (every 30s, queries
  EmailsView.getAllEmails, filters AWAITING_REVIEW with draftedAt older than 2
  minutes, calls EmailEntity.markEscalated).
- 2 HttpEndpoints: ResponderEndpoint at /api with emails ingest, approve,
  reject, list (filter client-side from getAllEmails), single, SSE stream, and
  three /api/metadata/* endpoints serving the YAML/MD from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html
  and /app/* -> static-resources/*.
- 1 service-setup Bootstrap scheduling InboxMonitor and StuckReviewMonitor.

Companion files:
- DraftReply(String subject, String body), Triage(boolean sensitive, String
  reason), EmailIn(String fromAddress, String subject, String body).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9165 and agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/inbound-emails.jsonl with 8 canned emails
  (mix of routine and sensitive; some carrying PII to exercise the sanitizer).
- src/main/resources/metadata/{eval-matrix.yaml,risk-survey.yaml,README.md}
  copies of the root files for the endpoint to serve from classpath.
- eval-matrix.yaml at project root with the 3 controls (S1, G1, H1) plus a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at project root, pre-filling sector, decisions, data.types,
  capability.*, model.*, subjects.children; marking deployer-specific fields
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at project root: pitch, component inventory, matrix cell,
  integration descriptor, how to run, the tabs, an ASCII architecture diagram,
  project layout, API contract, license. No governance-mechanisms section, no
  configuration section.
- src/main/resources/static-resources/index.html — single self-contained file
  (no ui/, no npm). Inline CSS + JS; runtime CDN imports for markdown and YAML
  acceptable. Five tabs Overview/Architecture/Risk Survey/Eval Matrix/App UI in
  the governance.html visual style (dark/yellow/Instrument Sans/dot-grid).

Generation workflow - see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding, inspect the environment for ANTHROPIC_API_KEY /
  OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set, default
  application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM - generate a MockModelProvider with per-agent canned/random
  outputs (TriageAgent -> Triage, ResponderAgent -> DraftReply; write
  src/main/resources/mock-responses/{triage-agent,responder-agent}.json with
  4-6 entries each). Sets model-provider = mock.
  (b) Name an existing env var - record the NAME in application.conf.
  (c) Point to an existing env file path - record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI - recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session - value lives in Claude session memory; passed
  to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Record only the
  REFERENCE - env-var name, file path, secrets URI - never the value.
- Bootstrap.java fails fast with a clear message naming the configured
  reference if it does not resolve at runtime; never echoes captured key.

Mock LLM provider per-agent shapes:
- TriageAgent -> Triage{sensitive: boolean, reason: short string}. Mock returns
  sensitive=true for emails mentioning refund/legal/complaint/account, else
  false.
- ResponderAgent -> DraftReply{subject: "Re: <original subject>", body: 2-4
  sentence acknowledgement referencing the request}.

Constraints - see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent (ResponderAgent) never silently downgraded to Agent.
- Lesson 4: workflow step timeouts set explicitly on agent-calling steps.
- Lesson 6: Optional<T> for every nullable lifecycle field on the Email row.
- Lesson 7: ResponderAgent ships its companion ResponderTasks.java.
- Lesson 8: verify model names are current before writing them.
- Lesson 9: run command is "/akka:build", never "mvn akka:run".
- Lesson 10: explicit port 9165 in application.conf.
- Lesson 11: no source-platform name anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: descriptive integration label "Runs out of the box", never tier codes.
- Lesson 23: no competitor brand names anywhere.
- Lesson 24: mermaid state-label CSS overrides + edge-label overflow:visible +
  transitionLabelColor #cccccc.
- Lesson 25: five-option key sourcing; never write the key to disk.
- Lesson 26: tab switching by data-tab/data-panel attribute; no zombie panels.
- emptyState() never calls commandContext().
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and 6.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
