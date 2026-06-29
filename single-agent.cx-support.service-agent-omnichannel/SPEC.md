# SPEC — service-agent-omnichannel

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Service Agent.
**One-line pitch:** An inbound customer message from any channel (WhatsApp, voice, web chat, or Facebook Messenger) triggers one AI agent that reads the case context, produces a structured reply, and either resolves the case or hands it off to a human support representative — all without a hard-coded dialogue script.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the cx-support domain. One `ServiceAgent` (AutonomousAgent) carries the entire decision of how to respond to a customer message; the surrounding components prepare its input, govern its output, and audit the outcome. Four governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw inbound message and the agent call — so the model never sees customer identifiers in the clear.
- A **before-agent-response guardrail** validates every outbound reply the agent drafts: correct structure, no prohibited phrases, channel-appropriate length.
- A **before-tool-call guardrail** gates every CRM case-write the agent requests: valid priority enum, required fields present, no write to a case owned by a different channel.
- A **HITL escalation path** (application-level) transfers the conversation to a live agent queue whenever the triage step or the customer's explicit request triggers escalation; the case enters `ESCALATED` state and the agent stops generating replies.

The blueprint shows that a single-agent pattern can carry both response generation and tool-use authorization under independent governance controls.

## 3. User-facing flows

The user opens the App UI tab.

1. The user selects an **Inbound channel** (WhatsApp / Voice / Web / Facebook) and a **Scenario** from a dropdown (billing dispute, technical fault, returns request, account closure) or pastes a custom message.
2. The user types a customer message in the **Customer message** textarea (or clicks **Load seeded example** to fill the channel, scenario, and message body).
3. The user clicks **Send message**. The UI POSTs to `/api/cases` and receives a `caseId`.
4. The card appears in the live list in `RECEIVED` state. Within ~1 s, it transitions to `SANITIZED` — the redacted message is visible in the card detail, with a small list of PII categories the sanitizer found.
5. Within ~1 s, the triage step classifies the message. The card shows a `CaseCategory` chip (billing / technical / returns / account / unknown).
6. Within ~10–30 s, the agent step completes. The card transitions to `HANDLING` then `REPLIED`. The reply appears: the agent's text, the channel it is formatted for, and a `ResolutionIntent` badge (RESOLVED / NEEDS_FOLLOW_UP / ESCALATE).
7. If `ResolutionIntent` is `ESCALATE`, the case transitions from `REPLIED` to `ESCALATED` and a human-queue chip appears on the card.
8. Within ~1 s of `REPLIED` (non-escalation), the eval step finishes. The card shows an **eval score** chip (1–5) and a one-line rationale about whether the reply was appropriate and complete.
9. The user can send another message; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `CaseEndpoint` | `HttpEndpoint` | `/api/cases/*` — ingest, list, get, SSE; serves `/api/metadata/*`. | — | `CaseEntity`, `CaseView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `CaseEntity` | `EventSourcedEntity` | Per-case lifecycle: received → sanitized → triaging → handling → replied → escalated / resolved. Source of truth. | `CaseEndpoint`, `MessageSanitizer`, `CaseWorkflow` | `CaseView` |
| `MessageSanitizer` | `Consumer` | Subscribes to `MessageReceived` events; redacts PII; calls `CaseEntity.attachSanitized`. | `CaseEntity` events | `CaseEntity` |
| `CaseWorkflow` | `Workflow` | One workflow per case. Steps: `awaitSanitizedStep` → `triageStep` → `handleStep` → `evalStep`. | started by `MessageSanitizer` once sanitized event lands | `ServiceAgent`, `CaseEntity` |
| `ServiceAgent` | `AutonomousAgent` | The one decision-making LLM. Receives case context in the task definition and the sanitized message as a task attachment; returns `AgentReply`. | invoked by `CaseWorkflow` | returns reply |
| `ReplyGuardrail` | supporting class | Registered on `ServiceAgent` via `before-agent-response`; validates reply structure and content policy. | `ServiceAgent` loop | — |
| `CrmWriteGuardrail` | supporting class | Registered on `ServiceAgent` via `before-tool-call`; gates CRM case-write tool calls. | `ServiceAgent` tool-call loop | — |
| `EscalationScorer` | supporting class | Deterministic rule-based scorer; fires in `evalStep` to score reply appropriateness. | invoked by `CaseWorkflow` | returns `EvalResult` |
| `CaseView` | `View` | Read model: one row per case for the UI. | `CaseEntity` events | `CaseEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
enum InboundChannel { WHATSAPP, VOICE, WEB, FACEBOOK }

enum CaseCategory { BILLING, TECHNICAL, RETURNS, ACCOUNT, UNKNOWN }

enum ResolutionIntent { RESOLVED, NEEDS_FOLLOW_UP, ESCALATE }

record InboundMessage(
    String caseId,
    InboundChannel channel,
    String customerId,
    String rawMessage,
    String scenario,
    Instant receivedAt
) {}

record SanitizedMessage(
    String redactedMessage,
    List<String> piiCategoriesFound
) {}

record CaseContext(
    String caseId,
    InboundChannel channel,
    CaseCategory category,
    String scenario
) {}

record AgentReply(
    ResolutionIntent resolutionIntent,
    String replyText,
    String channelFormat,        // "sms-short", "voice-ssml", "chat-markdown"
    List<String> crmWritesApplied,
    Instant decidedAt
) {}

record EvalResult(
    int score,              // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record Case(
    String caseId,
    Optional<InboundMessage> message,
    Optional<SanitizedMessage> sanitized,
    Optional<CaseCategory> category,
    Optional<AgentReply> reply,
    Optional<EvalResult> eval,
    CaseStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum CaseStatus {
    RECEIVED, SANITIZED, TRIAGING, HANDLING, REPLIED, ESCALATED, RESOLVED, FAILED
}
```

Events on `CaseEntity`: `MessageReceived`, `MessageSanitized`, `CaseTriaged`, `HandlingStarted`, `ReplySent`, `CaseEscalated`, `CaseResolved`, `EvaluationScored`, `CaseFailed`.

Every nullable lifecycle field on the `Case` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/cases` — body `{ channel, customerId, rawMessage, scenario }` → `{ caseId }`.
- `GET /api/cases` — list all cases, newest-first.
- `GET /api/cases/{id}` — one case.
- `GET /api/cases/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Service Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of cases (status pill + resolution badge + channel chip + age) and a right pane with the selected case's detail — inbound channel, sanitized message preview, triage category chip, agent reply, CRM writes applied, and eval score.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `MessageSanitizer` Consumer): redacts customer name, email, phone, account number, payment-card-like tokens, and postal address from the raw inbound message before any LLM call. Records which categories were found.
- **G1 — before-agent-response guardrail**: runs on every turn of `ServiceAgent`. Asserts (1) `replyText` is non-empty and within the per-channel character limit, (2) `channelFormat` is one of `{sms-short, voice-ssml, chat-markdown}`, (3) `resolutionIntent` is one of the allowed enum values, and (4) the reply contains none of the prohibited phrases (competitor names, price commitments outside authorised bounds, medical or legal advice). On failure returns a structured `invalid-response` error so the agent retries within its iteration budget.
- **G2 — before-tool-call guardrail**: runs before every CRM tool call the agent requests. Asserts (1) the tool name is in the approved list (`case.create`, `case.update`, `case.close`), (2) the `priority` field is one of `{LOW, NORMAL, HIGH, URGENT}`, (3) all required fields (`caseId`, `customerId`, `summary`) are non-empty, and (4) the case being written belongs to the channel the current message arrived on. Blocked calls return a `tool-rejected` error to the agent loop; the agent may correct and retry.
- **H1 — HITL escalation** (application-level): when the agent returns `resolutionIntent = ESCALATE`, the `handleStep` calls `CaseEntity.escalate(reason)`, which emits `CaseEscalated` and transitions the case to `ESCALATED`. No further agent turns occur; the case sits in the human-queue visible on the App UI. An operator handles it outside the system.

## 9. Agent prompts

- `ServiceAgent` → `prompts/service-agent.md`. The single decision-making LLM. System prompt instructs it to read the customer message, assess the scenario, produce a channel-formatted reply, and decide whether to resolve or escalate.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User sends a seeded billing-dispute message on the WEB channel; within 30 s a `REPLIED` card appears with a reply formatted as `chat-markdown`, an eval score chip, and at least one CRM write in `crmWritesApplied`.
2. **J2** — Agent's first response on a case contains a prohibited phrase; the `before-agent-response` guardrail rejects it; the second iteration produces a clean reply; the UI never displays the rejected draft.
3. **J3** — Agent requests a CRM write with an invalid `priority` value; the `before-tool-call` guardrail blocks it; the agent corrects the payload; the write succeeds; `crmWritesApplied` lists one entry.
4. **J4** — A message flagged as requiring a specialist triggers `resolutionIntent = ESCALATE`; the case transitions to `ESCALATED`; the App UI shows a human-queue chip; no further agent replies are generated.
5. **J5** — A message containing a customer email, phone number, and account number is submitted; the LLM call log shows only redacted tokens; `message.rawMessage` on the entity retains the raw text; `sanitized.piiCategoriesFound` lists `email`, `phone`, `account-number`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named service-agent-omnichannel demonstrating the single-agent × cx-support
cell. Runs out of the box (no external services; CRM writes are stubbed). Maven group
io.akka.samples. Maven artifact single-agent-cx-support-service-agent-omnichannel.
Java package io.akka.samples.serviceagent. Akka 3.6.0. HTTP port 9739.

Components to wire (exactly):

- 1 AutonomousAgent ServiceAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/service-agent.md>) and
  .capability(TaskAcceptance.of(HANDLE_CASE).maxIterationsPerTask(3)). The task receives
  case context (channel, category, scenario) as its instruction text and the sanitized
  customer message as a task ATTACHMENT (NOT as inline prompt text —
  TaskDef.attachment(name, contentBytes) is the canonical call).
  Output: AgentReply{resolutionIntent: ResolutionIntent, replyText: String,
  channelFormat: String, crmWritesApplied: List<String>, decidedAt: Instant}.
  The agent is configured with two guardrails:
  (a) a before-agent-response guardrail (ReplyGuardrail, see G1 in eval-matrix.yaml)
      registered via the agent's guardrail-configuration block. On rejection the agent
      loop retries within its 3-iteration budget.
  (b) a before-tool-call guardrail (CrmWriteGuardrail, see G2 in eval-matrix.yaml)
      registered via the agent's guardrail-configuration block. On rejection the agent
      corrects and retries the tool call within its iteration budget.

- 1 Workflow CaseWorkflow per caseId with four steps:
  * awaitSanitizedStep — polls CaseEntity.getCase every 1s; on case.sanitized().isPresent()
    advances to triageStep. WorkflowSettings.stepTimeout 15s.
  * triageStep — calls a deterministic CaseTriage.classify(sanitized.redactedMessage,
    case.message.scenario) to assign a CaseCategory; calls CaseEntity.recordTriage(category).
    WorkflowSettings.stepTimeout 5s.
  * handleStep — emits HandlingStarted, then calls componentClient.forAutonomousAgent(
    ServiceAgent.class, "agent-" + caseId).runSingleTask(
      TaskDef.instructions(formatCaseContext(case))
        .attachment("message.txt", case.sanitized.redactedMessage.getBytes())
    ) — returns a taskId, then forTask(taskId).result(HANDLE_CASE) to fetch the reply.
    On success calls CaseEntity.recordReply(reply). If reply.resolutionIntent == ESCALATE,
    also calls CaseEntity.escalate("agent-escalation").
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(CaseWorkflow::error).
  * evalStep — runs EscalationScorer deterministically over the recorded AgentReply:
    checks replyText is non-empty, channelFormat matches the case's channel, and
    resolutionIntent is consistent with the reply content (an ESCALATE intent on a
    routine billing query lowers the score). Emits EvaluationScored{score: 1-5,
    rationale: String}. WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity CaseEntity (one per caseId). State Case{caseId: String,
  message: Optional<InboundMessage>, sanitized: Optional<SanitizedMessage>,
  category: Optional<CaseCategory>, reply: Optional<AgentReply>,
  eval: Optional<EvalResult>, status: CaseStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. CaseStatus enum: RECEIVED,
  SANITIZED, TRIAGING, HANDLING, REPLIED, ESCALATED, RESOLVED, FAILED.
  Events: MessageReceived{message}, MessageSanitized{sanitized},
  CaseTriaged{category}, HandlingStarted{}, ReplySent{reply},
  CaseEscalated{reason}, CaseResolved{}, EvaluationScored{eval}, CaseFailed{reason}.
  Commands: ingest, attachSanitized, recordTriage, markHandling, recordReply,
  escalate, resolve, recordEvaluation, fail, getCase.
  emptyState() returns Case.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...)
  inside the event-applier.

- 1 Consumer MessageSanitizer subscribed to CaseEntity events; on MessageReceived runs
  a regex+heuristic redaction pipeline (email, phone, account-number, payment-card-like,
  postal address, person-name heuristic) over rawMessage, computes the list of categories
  found, builds SanitizedMessage, then calls CaseEntity.attachSanitized(sanitized).
  After attachSanitized lands, the same Consumer starts a CaseWorkflow with id =
  "case-" + caseId.

- 1 View CaseView with row type CaseRow (mirrors Case minus message.rawMessage — the
  audit log keeps the raw; the view holds the sanitized form for the UI). Table updater
  consumes CaseEntity events. ONE query getAllCases: SELECT * AS cases FROM case_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller
  filters client-side.

- 2 HttpEndpoints:
  * CaseEndpoint at /api with POST /cases (body {channel, customerId, rawMessage,
    scenario}; mints caseId; calls CaseEntity.ingest; returns {caseId}), GET /cases
    (list from getAllCases, sorted newest-first), GET /cases/{id} (one row), GET
    /cases/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* ->
    static-resources/*.

Companion files:

- CaseTasks.java declaring one Task<R> constant: HANDLE_CASE = Task.name("Handle support
  case").description("Read the attached customer message and produce an AgentReply with
  a resolution intent and channel-formatted reply text")
  .resultConformsTo(AgentReply.class). DO NOT skip this — the AutonomousAgent requires
  its companion Tasks class (Lesson 7).

- CaseTriage.java — pure deterministic classifier (no LLM). Input: redacted message
  String + scenario String. Output: CaseCategory. Rules: keyword matching against
  scenario plus a small regex set over message content. If no match: UNKNOWN.

- Domain records InboundMessage, SanitizedMessage, CaseContext, AgentReply, EvalResult,
  Case. Enums InboundChannel, CaseCategory, ResolutionIntent, CaseStatus.

- ReplyGuardrail.java implementing the before-agent-response hook. Reads the candidate
  AgentReply from the LLM response, runs the four checks listed in eval-matrix.yaml G1,
  and either passes through or returns Guardrail.reject(<structured-error>).

- CrmWriteGuardrail.java implementing the before-tool-call hook. Reads the candidate
  tool-call name and arguments, runs the four checks listed in eval-matrix.yaml G2,
  and either passes through or returns Guardrail.reject(<structured-error>).

- EscalationScorer.java — pure deterministic logic (no LLM). Inputs: AgentReply and
  CaseContext. Outputs: EvalResult. Scoring rubric documented in Javadoc.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9739 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
  The ServiceAgent.definition() binds the configured provider via
  .modelProvider("${akka.javasdk.agent.default}") or the per-agent override pattern
  from the akka-context docs.

- src/main/resources/sample-events/inbound-messages.jsonl with 4 seeded message sets:
  a billing-dispute (WEB), a technical-fault (WHATSAPP), a returns-request (FACEBOOK),
  and an account-closure (VOICE). Each contains 2–3 plausible PII strings so S1 has
  work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 4 controls (S1, G1, G2, H1) matching the
  mechanisms in Section 8 of this SPEC. Matching simplified_view list.
  No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = act-on-behalf
  (the agent composes and sends the reply directly), oversight.human_in_loop = true
  (HITL escalation path is live), failure.failure_modes including
  "inappropriate-reply", "missed-escalation", "pii-leakage-via-llm",
  "crm-write-with-bad-data", "channel-format-mismatch";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/service-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Service Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of case cards; right = selected-case detail with sanitized message,
  triage chip, reply text, CRM-writes list, and eval score chip).
  Browser title exactly: <title>Akka Sample: Service Agent</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        shape-correct outputs per agent (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf;
        /akka:build forwards the value from the Claude session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured
  key reference if it does not resolve at runtime. The message must not echo any
  captured key material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a
  per-task dispatch on the Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json, picks one entry pseudo-randomly
  per call (seedFor(caseId)), and deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    handle-support-case.json — 8 AgentReply entries covering all three ResolutionIntent
      values. Each entry has non-empty replyText, a valid channelFormat, at least one
      crmWritesApplied entry, and a realistic decidedAt. Plus 2 deliberately MALFORMED
      entries (one with channelFormat outside the enum; one with empty replyText) — the
      guardrail blocks both, exercising the retry path. The mock selects a malformed
      entry on the FIRST iteration of every 3rd case (modulo seed) so J2 is
      reproducible.
- A MockModelProvider.seedFor(caseId) helper makes per-case selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ServiceAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion CaseTasks.java MUST
  exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout
  (handleStep 60s, awaitSanitizedStep 15s, triageStep 5s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Case row record is Optional<T>. The
  view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: CaseTasks.java with HANDLE_CASE = Task.name(...).description(...)
  .resultConformsTo(AgentReply.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9739 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in
  narrative, marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS
  overrides (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only
  the reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by
  NodeList index. The DOM contains exactly five <section class="tab-panel"> elements —
  Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (ServiceAgent). The
  triage step (CaseTriage.java) and the eval step (EscalationScorer.java) are
  deterministic rule-based components — no LLM calls — keeping the pattern honest.
- The customer message is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated handleStep uses TaskDef.attachment(...) and not
  string interpolation into the instruction text.
- The before-agent-response guardrail and the before-tool-call guardrail are both
  wired via the agent's guardrail-configuration block, not as external checks. Lesson
  1's AutonomousAgent contract is the authoritative reference.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
  Per Lesson 25, /akka:specify handles the key during generation.
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
