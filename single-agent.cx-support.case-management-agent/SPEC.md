# SPEC — case-management-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** CaseManagementAgent.
**One-line pitch:** An inbound customer message arrives at any hour; one AI agent reads the sanitized message, classifies it, and either creates a new case or updates an existing one — with two guardrails watching the action before it reaches the CRM store.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the cx-support domain. One `SupportAgent` (AutonomousAgent) carries the entire decision — classify the message, choose an action (CREATE / UPDATE / ESCALATE / CLOSE), fill the required fields, and invoke the matching CRM tool. Three governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw inbound message and the agent call — so the model never sees the customer's personal identifiers.
- A **before-agent-response guardrail** validates the agent's proposed action on every turn: the action type is in the allowed set, required fields (category, priority, tier) are present, and the case id is provided for UPDATE/ESCALATE/CLOSE actions. A malformed proposal triggers a retry inside the same task.
- A **before-tool-call guardrail** runs immediately before each CRM tool invocation: verifies the target case exists for mutations, checks escalation-path policy (e.g., TIER_1 cannot escalate directly to TIER_3), and rejects writes that violate mandatory-field constraints. This guard runs after the agent has already produced a well-formed action — it is the last check before the write lands.

The blueprint shows that two independent guardrails at two different hooks can each enforce a distinct risk surface without duplicating logic.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a customer message into the **Message** textarea (or picks one of three seeded examples — a billing dispute, a technical outage report, an account-access request).
2. The user picks a **customer id** and optionally an **existing case id** (to test the UPDATE path), then clicks **Send message**. The UI POSTs to `/api/cases/messages` and receives a `messageId`.
3. The inbound message card appears in `RECEIVED` state. Within ~1 s it transitions to `SANITIZED` — a small row shows how many PII categories were found.
4. Within ~10–30 s the agent completes its turn. The card transitions to `AGENT_ACTING` then `ACTION_APPLIED`. For a CREATE action a new case row appears in the case list. For UPDATE/ESCALATE/CLOSE the matching case row refreshes.
5. The case row shows: case id, category chip, priority chip, tier, status, last-action summary, and the eval score chip (1–5).
6. The user can send another message or click any case row to open the full detail panel: sanitized message, agent's reasoning, action taken, CRM record snapshot, eval rationale.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `CaseEndpoint` | `HttpEndpoint` | `/api/cases/*` and `/api/messages/*` — submit message, list cases, get case, SSE; serves `/api/metadata/*`. | — | `CaseEntity`, `CaseView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `CaseEntity` | `EventSourcedEntity` | Per-case lifecycle: open → in-progress → escalated → resolved. Source of truth. | `CaseEndpoint`, `MessageSanitizer`, `CaseWorkflow` | `CaseView` |
| `MessageSanitizer` | `Consumer` | Subscribes to `MessageReceived` events; redacts PII; calls `CaseEntity.attachSanitized`. | `CaseEntity` events | `CaseEntity` |
| `CaseWorkflow` | `Workflow` | One workflow per inbound message. Steps: `awaitSanitizedStep` → `agentStep` → `evalStep`. | started by `MessageSanitizer` once sanitized event lands | `SupportAgent`, `CaseEntity` |
| `SupportAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the sanitized message plus open-case context as task instructions; invokes CRM tools via `before-tool-call`-guarded tool calls; returns `CaseAction`. | invoked by `CaseWorkflow` | returns action; invokes CRM tools |
| `ActionGuardrail` | supporting class | before-agent-response hook: validates `CaseAction` structure before the agent loop accepts it. | wired on `SupportAgent` | — |
| `ToolCallGuardrail` | supporting class | before-tool-call hook: enforces CRM write policy before each tool invocation. | wired on `SupportAgent` | — |
| `ActionEvaluator` | supporting class | Deterministic scorer; runs in `evalStep`; no LLM call. | invoked by `CaseWorkflow` | `EvalResult` |
| `CaseView` | `View` | Read model: one row per case for the UI. | `CaseEntity` events | `CaseEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record InboundMessage(
    String messageId,
    String customerId,
    Optional<String> existingCaseId,
    String rawText,
    Channel channel,
    Instant receivedAt
) {}
enum Channel { WEB_CHAT, EMAIL, PHONE_TRANSCRIPT, API }

record SanitizedMessage(
    String redactedText,
    List<String> piiCategoriesFound
) {}

record CaseAction(
    ActionType actionType,
    Optional<String> targetCaseId,       // required for UPDATE/ESCALATE/CLOSE
    String category,
    Priority priority,
    Tier tier,
    String summary,
    String agentReasoning
) {}
enum ActionType { CREATE, UPDATE, ESCALATE, CLOSE }
enum Priority   { LOW, MEDIUM, HIGH, CRITICAL }
enum Tier       { TIER_1, TIER_2, TIER_3 }

record CrmRecord(
    String caseId,
    String customerId,
    String category,
    Priority priority,
    Tier tier,
    CaseStatus status,
    String summary,
    List<String> messageIds,
    Instant openedAt,
    Optional<Instant> resolvedAt
) {}
enum CaseStatus { OPEN, IN_PROGRESS, ESCALATED, RESOLVED }

record EvalResult(
    int score,           // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record CaseRecord(
    String caseId,
    Optional<InboundMessage> lastMessage,
    Optional<SanitizedMessage> sanitizedMessage,
    Optional<CaseAction> lastAction,
    Optional<CrmRecord> crmRecord,
    Optional<EvalResult> eval,
    CaseStatus status,
    Instant createdAt,
    Optional<Instant> resolvedAt
) {}

enum CaseStatus { OPEN, IN_PROGRESS, ESCALATED, RESOLVED }
```

Events on `CaseEntity`: `MessageReceived`, `MessageSanitized`, `AgentActing`, `ActionApplied`, `EvaluationScored`, `CaseFailed`.

Every nullable lifecycle field on the `CaseRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/messages` — body `{ customerId, existingCaseId?, rawText, channel }` → `{ messageId, caseId }`.
- `GET /api/cases` — list all cases, newest-first.
- `GET /api/cases/{id}` — one case.
- `GET /api/cases/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Case Management Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live case list (status pill + priority chip + category + tier + age) and a right pane with the selected case's detail — sanitized message, agent reasoning, action taken, CRM record snapshot, and eval-score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `MessageSanitizer` Consumer): redacts emails, phone numbers, government identifiers, payment-card-like tokens, person names, postal addresses, and account-like identifiers from the raw inbound message before any LLM call. Records which categories were found.
- **G1 — before-agent-response guardrail**: runs on every turn of `SupportAgent`. Asserts the candidate response is a well-formed `CaseAction`, the `actionType` is in `{CREATE, UPDATE, ESCALATE, CLOSE}`, `priority` and `tier` are in their respective enums, and `targetCaseId` is present for mutating actions. On failure, returns a structured `invalid-response` error so the task retries within its iteration budget.
- **G2 — before-tool-call guardrail**: runs immediately before each CRM tool invocation inside `SupportAgent`. Verifies the target case exists for UPDATE/ESCALATE/CLOSE, enforces the escalation-path policy (TIER_1 → TIER_2 only; TIER_2 → TIER_3 only), and rejects any write missing mandatory CRM fields. Blocking: a rejected tool call prevents execution; the agent receives the rejection reason and may revise its tool arguments.

## 9. Agent prompts

- `SupportAgent` → `prompts/support-agent.md`. The single decision-making LLM. System prompt instructs it to read the sanitized customer message, choose an action type, fill all required fields, and invoke the correct CRM tool.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User sends a billing-dispute message; within 30 s a new case appears with category=billing, priority=MEDIUM, tier=TIER_1, and an eval score chip.
2. **J2** — Agent's first response is missing the required `priority` field — the `before-agent-response` guardrail rejects it; the second iteration produces a valid action; the UI never displays the malformed response.
3. **J3** — Agent attempts to escalate a TIER_1 case directly to TIER_3 — the `before-tool-call` guardrail blocks the write and returns a policy-violation reason; the agent retises with a TIER_2 escalation.
4. **J4** — A message containing `john.doe@example.com` and `card 4111 1111 1111 1111` is submitted; the LLM call log shows only `[REDACTED-EMAIL]` and `[REDACTED-PAYMENT-CARD]`; the entity's `lastMessage.rawText` retains the raw text for audit.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named case-management-agent demonstrating the single-agent × cx-support cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-cx-support-case-management-agent. Java package
io.akka.samples.casemanagementagent. Akka 3.6.0. HTTP port 9695.

Components to wire (exactly):

- 1 AutonomousAgent SupportAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/support-agent.md>) and
  .capability(TaskAcceptance.of(HANDLE_MESSAGE).maxIterationsPerTask(3)).
  The task receives the sanitized message text plus any matching open-case summary
  as its instruction text. Output: CaseAction{actionType: ActionType
  (CREATE/UPDATE/ESCALATE/CLOSE), targetCaseId: Optional<String>, category: String,
  priority: Priority, tier: Tier, summary: String, agentReasoning: String}.
  The agent is configured with a before-agent-response guardrail (ActionGuardrail, see G1
  in eval-matrix.yaml) and a before-tool-call guardrail (ToolCallGuardrail, see G2).
  Both are registered via the agent's guardrail-configuration block. On guardrail rejection
  the agent loop retries within its 3-iteration budget.

- 1 Workflow CaseWorkflow per messageId with three steps:
  * awaitSanitizedStep — polls CaseEntity.getCase every 1s; on case.sanitizedMessage()
    .isPresent() advances to agentStep. WorkflowSettings.stepTimeout 15s.
  * agentStep — emits AgentActing, then calls componentClient.forAutonomousAgent(
    SupportAgent.class, "agent-" + messageId).runSingleTask(
      TaskDef.instructions(formatMessageContext(caseRecord))
    ) — returns a taskId, then forTask(taskId).result(HANDLE_MESSAGE) to fetch the action.
    On success calls CaseEntity.applyAction(action). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(2).failoverTo(CaseWorkflow::error).
  * evalStep — runs deterministic ActionEvaluator (NOT an LLM call) over the recorded action:
    checks that summary is non-empty, reasoning is present, priority assignment is
    consistent with the category (CRITICAL priority on a billing-inquiry is suspect unless
    category is billing-fraud), and action type is appropriate for the case status.
    Emits EvaluationScored{score: 1-5, rationale: String}. WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity CaseEntity (one per caseId). State CaseRecord{caseId: String,
  lastMessage: Optional<InboundMessage>, sanitizedMessage: Optional<SanitizedMessage>,
  lastAction: Optional<CaseAction>, crmRecord: Optional<CrmRecord>,
  eval: Optional<EvalResult>, status: CaseStatus, createdAt: Instant,
  resolvedAt: Optional<Instant>}. CaseStatus enum: OPEN, IN_PROGRESS, ESCALATED, RESOLVED.
  Events: MessageReceived{message}, MessageSanitized{sanitized}, AgentActing{},
  ActionApplied{action, crmRecord}, EvaluationScored{eval}, CaseFailed{reason}.
  Commands: receiveMessage, attachSanitized, markActing, applyAction, recordEvaluation,
  fail, getCase. emptyState() returns CaseRecord.initial("") with no commandContext()
  reference (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 Consumer MessageSanitizer subscribed to CaseEntity events; on MessageReceived runs
  a regex+heuristic redaction pipeline (emails, phone numbers, payment-card-like tokens,
  SSN-like, postal addresses, person-name heuristic, account-id-like tokens) over rawText,
  computes the list of categories found, builds SanitizedMessage, then calls
  CaseEntity.attachSanitized(sanitized). After attachSanitized lands, the same Consumer
  starts a CaseWorkflow with id = "case-" + messageId.

- 1 View CaseView with row type CaseRow (mirrors CaseRecord minus lastMessage.rawText —
  the audit log keeps the raw; the view holds the sanitized form for the UI). Table updater
  consumes CaseEntity events. ONE query getAllCases: SELECT * AS cases FROM case_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * CaseEndpoint at /api with POST /messages (body
    {customerId, existingCaseId?, rawText, channel}; mints messageId; determines
    caseId (new UUID for CREATE path or existingCaseId for others); calls
    CaseEntity.receiveMessage; returns {messageId, caseId}), GET /cases (list from
    getAllCases, sorted newest-first), GET /cases/{id} (one row), GET /cases/sse
    (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* ->
    static-resources/*.

Companion files:

- CaseTasks.java declaring one Task<R> constant: HANDLE_MESSAGE = Task
  .name("Handle customer message")
  .description("Read the sanitized message and open-case context, choose an action, and return a CaseAction")
  .resultConformsTo(CaseAction.class). DO NOT skip this — the AutonomousAgent
  requires its companion Tasks class (Lesson 7).

- Domain records InboundMessage, SanitizedMessage, CaseAction, CrmRecord, EvalResult,
  CaseRecord and enums Channel, ActionType, Priority, Tier, CaseStatus.

- ActionGuardrail.java implementing the before-agent-response hook. Reads the candidate
  CaseAction from the LLM response, runs the checks listed in eval-matrix.yaml G1,
  and either passes the response through or returns Guardrail.reject(<structured-error>)
  to force the agent loop to retry.

- ToolCallGuardrail.java implementing the before-tool-call hook. Checks each CRM tool
  invocation against the policy rules listed in eval-matrix.yaml G2. Returns
  Guardrail.reject(<policy-violation-reason>) on violations; passes compliant calls through.

- ActionEvaluator.java — pure deterministic logic (no LLM). Inputs: CaseAction and the
  original InboundMessage channel + CaseStatus. Outputs: EvalResult. Scoring rubric
  documented in Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9695 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The SupportAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/messages.jsonl with 3 seeded inbound messages:
  a billing dispute (contains email + card number for PII testing), a technical outage
  report (contains phone number + name), and an account-access request (contains
  account id + address). Each triggers a different action path.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (S1, G1, G2) matching the
  mechanisms in Section 8. Matching simplified_view list.

- risk-survey.yaml at the project root pre-filled for cx-support domain.

- prompts/support-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Case Management Agent",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms
  section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live case list; right = selected-case detail with sanitized message, agent
  reasoning, action taken, CRM snapshot, and eval-score chip).
  Browser title exactly: <title>Akka Sample: Case Management Agent</title>. No subtitle
  on the Overview tab.

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

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java with per-task dispatch on the Task<R> id. Each branch
  reads src/main/resources/mock-responses/<task-id>.json and picks one entry
  pseudo-randomly per call (seedFor(messageId)).
- Per-task mock-response shapes for THIS blueprint:
    handle-customer-message.json — 8 CaseAction entries covering all four ActionType
    values. Each entry has a non-empty summary and agentReasoning, a valid category,
    priority, and tier. Plus 2 deliberately MALFORMED entries (one missing the priority
    field; one with an actionType value outside the enum) — ActionGuardrail blocks both,
    exercising the retry path. The mock selects a malformed entry on the FIRST iteration
    of every 3rd message (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(messageId) helper makes per-message selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. SupportAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion CaseTasks.java MUST exist.
- Lesson 4: every workflow step has explicit stepTimeout (agentStep 60s, awaitSanitizedStep
  15s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on CaseRecord is Optional<T>.
- Lesson 7: CaseTasks.java with HANDLE_MESSAGE Task is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9695 declared explicitly in application.conf.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words.
- Lesson 24: static-resources/index.html includes mermaid CSS overrides and themeVariables.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching by data-tab / data-panel attribute only; exactly five
  <section class="tab-panel"> elements.
- The single-agent invariant: exactly ONE AutonomousAgent (SupportAgent). ActionEvaluator
  is deterministic (no LLM call).
- ActionGuardrail is wired via before-agent-response; ToolCallGuardrail via before-tool-call.
  Both are registered on SupportAgent's guardrail-configuration block, not as external
  checks that run after the agent returns.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
