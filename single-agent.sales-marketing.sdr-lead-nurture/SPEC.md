# SPEC — sdr-lead-nurture

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** SDR Agent.
**One-line pitch:** An inbound lead arrives; one AI agent handles the entire nurture conversation — answering product questions, working through objections, qualifying intent, and booking a meeting when the lead is ready — while three governance mechanisms ensure brand safety, protect write-action tool calls, and keep contact details out of the prompt context.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the sales-marketing domain. One `SdrAgent` (AutonomousAgent) carries every turn of the lead-nurture conversation; the surrounding components only prepare its input, audit its output, and guard its tool calls. Three governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw lead-record ingest and the first agent turn — so the model never sees raw email addresses, phone numbers, or personal identifiers in the contact record.
- A **before-agent-response guardrail** validates every candidate outbound message for brand safety: no unapproved pricing commitments, no competitor comparisons, no prohibited topics, and well-formed JSON envelope. A violating draft triggers a retry inside the same task.
- A **before-tool-call guardrail** inspects every write-action tool call (calendar booking, CRM status update) before the call is dispatched: checks the slot is within allowed booking hours, the meeting duration is in range, and the CRM field values are within allowed enumerations. A blocked tool call returns a structured error to the agent loop so it can recover.

The blueprint shows that single-agent does not mean "ungoverned" — three independent checks gate the one decision-making LLM.

## 3. User-facing flows

The user opens the App UI tab.

1. The user clicks **Ingest lead** and fills in the lead form: first name, last name, company, job title, email, inbound channel (website-chat / email / event / referral), and an initial message the lead sent.
2. The user clicks **Start engagement**. The UI POSTs to `/api/leads` and receives a `leadId`.
3. The card appears in the live list in `RECEIVED` state. Within ~1 s it transitions to `SANITIZED` — the redacted contact record is visible, with a list of PII categories found.
4. Within ~2 s, the first agent turn completes. The card transitions to `ENGAGING`. The right pane shows the agent's first reply (the outbound message that passed the brand-safety guardrail).
5. The user (acting as the lead) types a follow-up message in the conversation thread and clicks **Send**. The card updates as the agent responds each turn.
6. When the agent determines the lead is qualified and ready, it calls the `BookMeeting` tool. The `before-tool-call` guardrail checks the proposed slot. On success, the card transitions to `MEETING_BOOKED`. The right pane shows the meeting confirmation details and the updated CRM status.
7. Alternatively, the agent can call `DismissLead` (unqualified) or `HandoffToAE` (high-value, needs human). Those terminal actions transition the entity accordingly.
8. Throughout the conversation an on-decision evaluator scores the quality of the qualification — is the agent asking discovery questions, is it pushing to close too early, is the tone on-brand?

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `LeadEndpoint` | `HttpEndpoint` | `/api/leads/*` — ingest, reply, list, get, SSE; serves `/api/metadata/*`. | — | `LeadEntity`, `LeadView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `LeadEntity` | `EventSourcedEntity` | Per-lead lifecycle: received → sanitized → engaging → meeting booked / dismissed / handed off. Source of truth. | `LeadEndpoint`, `LeadSanitizer`, `LeadWorkflow` | `LeadView` |
| `LeadSanitizer` | `Consumer` | Subscribes to `LeadReceived` events; strips PII from the contact record; calls `LeadEntity.attachSanitized`. | `LeadEntity` events | `LeadEntity` |
| `LeadWorkflow` | `Workflow` | One workflow per lead. Steps: `awaitSanitizedStep` → `engageStep` → `closeStep`. | started by `LeadSanitizer` once sanitized event lands | `SdrAgent`, `LeadEntity` |
| `SdrAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the sanitized lead record + conversation history as task context; returns an `AgentTurn` (outbound message + optional tool call). | invoked by `LeadWorkflow` | returns `AgentTurn` |
| `ReplyGuardrail` | supporting class | `before-agent-response` hook on `SdrAgent`. Validates candidate outbound messages for brand safety. | wired to `SdrAgent` | — |
| `BookingGuardrail` | supporting class | `before-tool-call` hook on `SdrAgent`. Validates calendar-booking and CRM-mutation calls before execution. | wired to `SdrAgent` | — |
| `QualityScorer` | supporting class | Deterministic rule-based evaluator. Scores the qualification quality after each close action. | invoked by `LeadWorkflow.closeStep` | returns `QualityScore` |
| `LeadView` | `View` | Read model: one row per lead for the UI. | `LeadEntity` events | `LeadEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record LeadContact(
    String leadId,
    String firstName,
    String lastName,
    String company,
    String jobTitle,
    String email,
    String phone,           // may be blank
    InboundChannel channel,
    String initialMessage,
    Instant receivedAt
) {}
enum InboundChannel { WEBSITE_CHAT, EMAIL, EVENT, REFERRAL }

record SanitizedContact(
    String redactedRecord,          // JSON with PII replaced by tokens
    List<String> piiCategoriesFound
) {}

record ConversationTurn(
    String turnId,
    TurnRole role,
    String message,
    Instant sentAt
) {}
enum TurnRole { LEAD, AGENT }

record AgentTurn(
    String message,
    Optional<ToolCall> toolCall,
    Instant decidedAt
) {}

record ToolCall(
    ToolName tool,
    Map<String, Object> params
) {}
enum ToolName { BOOK_MEETING, UPDATE_CRM_STATUS, DISMISS_LEAD, HANDOFF_TO_AE }

record MeetingBooking(
    String slot,           // ISO-8601 datetime
    int durationMinutes,
    String accountExecutiveId,
    String zoomLink
) {}

record CrmUpdate(
    LeadStatus newStatus,
    String note
) {}

record QualityScore(
    int score,              // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record Lead(
    String leadId,
    Optional<LeadContact> contact,
    Optional<SanitizedContact> sanitized,
    List<ConversationTurn> conversation,
    Optional<MeetingBooking> booking,
    Optional<QualityScore> quality,
    LeadStatus status,
    Instant createdAt,
    Optional<Instant> closedAt
) {}

enum LeadStatus {
    RECEIVED, SANITIZED, ENGAGING, MEETING_BOOKED, DISMISSED, HANDED_OFF, FAILED
}
```

Events on `LeadEntity`: `LeadReceived`, `LeadSanitized`, `EngagementStarted`, `TurnRecorded`, `MeetingBooked`, `LeadDismissed`, `HandedOffToAE`, `QualityScoredEvent`, `LeadFailed`.

Every nullable lifecycle field on the `Lead` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/leads` — body `{ firstName, lastName, company, jobTitle, email, phone, channel, initialMessage }` → `{ leadId }`.
- `POST /api/leads/{id}/reply` — body `{ message }` → `202`. Appends a `LEAD` turn; triggers the next agent turn.
- `GET /api/leads` — list all leads, newest-first.
- `GET /api/leads/{id}` — one lead.
- `GET /api/leads/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: SDR Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of leads (status pill + lead name + company + age) and a right pane with the selected lead's detail — contact summary, sanitized record preview, conversation thread, booking card, and quality score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `LeadSanitizer` Consumer): strips email addresses, phone numbers, personal names, postal addresses, and account-like identifiers from the raw contact record before any LLM call. Records which categories were found.
- **G1 — before-agent-response guardrail**: runs on every turn of `SdrAgent`. Asserts the candidate reply message is non-empty, contains no unapproved pricing commitments (pattern-matched against a deny list), references no competitor brand names (pattern-matched deny list), and has a parseable `AgentTurn` envelope. On failure, returns a structured `brand-violation` error to the agent loop so it retries within the iteration budget.
- **G2 — before-tool-call guardrail**: runs before any `ToolCall` dispatched by `SdrAgent`. For `BOOK_MEETING`: asserts the proposed `slot` falls within weekday 08:00–17:00 in the configured timezone, and `durationMinutes` is 15, 30, or 45. For `UPDATE_CRM_STATUS`: asserts the target `newStatus` is a member of `LeadStatus`. For any other tool: allows through. On failure, returns a structured `tool-blocked` error.

## 9. Agent prompts

- `SdrAgent` → `prompts/sdr-agent.md`. The single decision-making LLM. System prompt instructs it to nurture the lead through the qualification funnel: acknowledge the inbound message, ask discovery questions, address objections, and — when intent is clear — call `BOOK_MEETING` or `HANDOFF_TO_AE`. It must never reveal internal tooling, never commit to pricing not in the product catalogue, and never mention competitors.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — A lead arrives via the seeded dataset; within 30 s the first agent reply is visible in the conversation thread with a quality score chip.
2. **J2** — The agent's first candidate reply on a conversation contains a pricing commitment that is not in the approved list — the `before-agent-response` guardrail rejects it; the second iteration produces a compliant reply; the non-compliant draft never lands in the entity log.
3. **J3** — The agent attempts to book a meeting at 07:00 on a Saturday — the `before-tool-call` guardrail blocks the call with a `tool-blocked` error; the agent proposes a weekday slot instead.
4. **J4** — Lead contact details (email, phone) submitted with the inbound record never appear in the LLM call log; only the redacted form does.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named sdr-lead-nurture demonstrating the single-agent × sales-marketing cell.
Maven group io.akka.samples. Maven artifact single-agent-sales-marketing-sdr-lead-nurture.
Java package io.akka.samples.sdragent. Akka 3.6.0. HTTP port 9409.

Components to wire (exactly):

- 1 AutonomousAgent SdrAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/sdr-agent.md>) and
  .capability(TaskAcceptance.of(SDR_ENGAGE).maxIterationsPerTask(5)). The task receives
  the sanitized lead record and conversation history as its instruction context. Output:
  AgentTurn{message: String, toolCall: Optional<ToolCall>, decidedAt: Instant}.
  The agent is configured with two guardrails:
  - ReplyGuardrail (before-agent-response, see G1 in eval-matrix.yaml) registered via the
    agent's guardrail-configuration block. On rejection the agent loop retries within its
    5-iteration budget.
  - BookingGuardrail (before-tool-call, see G2 in eval-matrix.yaml) registered via the
    agent's tool-call-guardrail-configuration block. On rejection returns a structured
    tool-blocked error to the agent loop so it recovers without failing the workflow step.

- 1 Workflow LeadWorkflow per leadId with three steps:
  * awaitSanitizedStep — polls LeadEntity.getLead every 1s; on lead.sanitized().isPresent()
    advances to engageStep. WorkflowSettings.stepTimeout 15s.
  * engageStep — emits EngagementStarted, then calls componentClient.forAutonomousAgent(
    SdrAgent.class, "sdr-" + leadId).runSingleTask(
      TaskDef.instructions(formatLeadContext(lead))
    ) — returns a taskId, then forTask(taskId).result(SDR_ENGAGE) to fetch the AgentTurn.
    Records the turn via LeadEntity.recordTurn(turn). If the AgentTurn carries a ToolCall,
    dispatches it (see tool dispatch below). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(2).failoverTo(LeadWorkflow::error).
  * closeStep — runs QualityScorer over the complete conversation and the close action.
    Emits QualityScoredEvent{score, rationale}. WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  Tool dispatch inside engageStep:
  - BOOK_MEETING → calls BookingGuardrail check (already wired as before-tool-call), then
    calls a lightweight in-process CalendarStub.book(params) → returns MeetingBooking.
    Calls LeadEntity.recordBooking(booking).
  - UPDATE_CRM_STATUS → calls LeadEntity.updateStatus(newStatus, note).
  - DISMISS_LEAD → calls LeadEntity.dismiss(reason).
  - HANDOFF_TO_AE → calls LeadEntity.handoffToAE(note).

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity LeadEntity (one per leadId). State Lead{leadId: String,
  contact: Optional<LeadContact>, sanitized: Optional<SanitizedContact>,
  conversation: List<ConversationTurn>, booking: Optional<MeetingBooking>,
  quality: Optional<QualityScore>, status: LeadStatus,
  createdAt: Instant, closedAt: Optional<Instant>}. LeadStatus enum: RECEIVED,
  SANITIZED, ENGAGING, MEETING_BOOKED, DISMISSED, HANDED_OFF, FAILED.
  Events: LeadReceived{contact}, LeadSanitized{sanitized}, EngagementStarted{},
  TurnRecorded{turn}, MeetingBooked{booking}, LeadDismissed{reason},
  HandedOffToAE{note}, QualityScoredEvent{quality}, LeadFailed{reason}.
  Commands: receive, attachSanitized, startEngagement, recordTurn, recordBooking,
  updateStatus, dismiss, handoffToAE, recordQuality, fail, getLead.
  emptyState() returns Lead.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...)
  inside the event-applier.

- 1 Consumer LeadSanitizer subscribed to LeadEntity events; on LeadReceived runs a
  regex+heuristic redaction pipeline (emails, phone numbers, person-name heuristic,
  postal addresses, account-id-like tokens) over the serialized contact record, computes
  the list of categories found, builds SanitizedContact, then calls
  LeadEntity.attachSanitized(sanitized). After attachSanitized lands, the same Consumer
  starts a LeadWorkflow with id = "lead-" + leadId.

- 1 View LeadView with row type LeadRow (mirrors Lead minus contact.email and
  contact.phone — PII is retained on the entity for audit; the view holds the sanitized
  form). Table updater consumes LeadEntity events. ONE query getAllLeads:
  SELECT * AS leads FROM lead_view. No WHERE status filter — Akka cannot auto-index
  enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * LeadEndpoint at /api with:
    POST /leads (body {firstName, lastName, company, jobTitle, email, phone, channel,
      initialMessage}; mints leadId; calls LeadEntity.receive; returns {leadId}),
    POST /leads/{id}/reply (body {message}; appends TurnRecorded{LEAD}; triggers next
      engageStep),
    GET /leads (list from getAllLeads, sorted newest-first),
    GET /leads/{id} (one row),
    GET /leads/sse (Server-Sent Events forwarded from the view's stream-updates),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- LeadTasks.java declaring one Task<R> constant: SDR_ENGAGE = Task.name("Engage lead")
  .description("Nurture the inbound lead through qualification; reply to their message
  or call a close-action tool").resultConformsTo(AgentTurn.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records LeadContact, SanitizedContact, ConversationTurn, TurnRole, AgentTurn,
  ToolCall, ToolName, MeetingBooking, CrmUpdate, QualityScore, Lead, LeadStatus,
  InboundChannel.

- ReplyGuardrail.java implementing the before-agent-response hook. Reads the candidate
  AgentTurn from the LLM response, checks: (1) message is non-empty, (2) no unapproved
  pricing patterns (regex against a deny list loaded from
  src/main/resources/brand-rules/pricing-deny-list.txt), (3) no competitor brand names
  (deny list from src/main/resources/brand-rules/competitor-deny-list.txt).
  Returns Guardrail.reject(<brand-violation>) on failure.

- BookingGuardrail.java implementing the before-tool-call hook. On BOOK_MEETING: asserts
  slot is a weekday, hour is between 08:00 and 17:00 in configured timezone, and
  durationMinutes is in {15, 30, 45}. On UPDATE_CRM_STATUS: asserts newStatus is a
  valid LeadStatus enum value. Other tools: pass through.
  Returns Guardrail.reject(<tool-blocked>) on failure.

- QualityScorer.java — pure deterministic logic (no LLM). Inputs: List<ConversationTurn>
  and the close action (ToolCall). Scoring rubric: +1 if at least 2 discovery questions
  were asked, +1 if objections were addressed, +1 if the tone was on-brand throughout
  (heuristic: no all-caps sentences, no exclamation-only sentences), +1 if the close
  action is proportional to conversation length, -1 if the agent pushed to BOOK_MEETING
  in fewer than 2 turns. Score clipped to 1..5. Documented in Javadoc on the class.

- CalendarStub.java — in-process stub (no external network call). Accepts a MeetingBooking
  params map, validates the slot format, and returns a MeetingBooking with a synthetic
  zoomLink. This keeps the blueprint self-contained; a deployer replaces it with a real
  calendar API client.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9409 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
  The SdrAgent.definition() binds the configured provider via the per-agent override
  pattern from the akka-context docs.

- src/main/resources/sample-events/seed-leads.jsonl with 4 seeded leads across the
  four inbound channels. Each includes realistic first/last name, company, job title,
  email, phone, and an initial message. Two leads are mid-funnel (likely to book), one
  is low-intent (leads to DISMISSED), one is enterprise (leads to HANDOFF_TO_AE).

- src/main/resources/brand-rules/pricing-deny-list.txt — a list of prohibited pricing
  phrases, one per line, e.g. "free forever", "guaranteed ROI", "half the price".

- src/main/resources/brand-rules/competitor-deny-list.txt — a list of competitor brand
  names to never mention, one per line.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies
  of the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (S1, G1, G2) matching the
  mechanisms in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root pre-filled for sales-marketing domain.

- prompts/sdr-agent.md loaded as the agent system prompt.

- README.md at the project root matching the blueprint README.

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of lead cards; right = selected-lead detail with contact summary,
  sanitized record, conversation thread, booking card, quality score chip).
  Browser title exactly: <title>Akka Sample: SDR Agent</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java per the standard pattern. Per-task mock-response shapes:
    sdr-engage.json — 8 AgentTurn entries covering mid-funnel reply, discovery question,
    objection handling, booking proposal, handoff proposal, and dismissal. Each entry has
    a realistic message text and an optional toolCall block. Plus 2 deliberately non-
    compliant entries (one with a pricing phrase from the deny list; one with a competitor
    name) — ReplyGuardrail blocks both, exercising the retry path. Plus 1 entry with a
    BOOK_MEETING toolCall where the slot is outside allowed hours — BookingGuardrail
    blocks it, exercising the before-tool-call path.
- MockModelProvider.seedFor(leadId) makes per-lead selection deterministic.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. SdrAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion LeadTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (engageStep
  60s, awaitSanitizedStep 15s, closeStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Lead row record is Optional<T>.
- Lesson 7: LeadTasks.java with SDR_ENGAGE = Task.name(...).description(...)
  .resultConformsTo(AgentTurn.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9409 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs with an in-process calendar stub" — never T1/T2/T3/T4
  in any user-visible string.
- Lesson 23: no forbidden words.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  and the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (SdrAgent). The quality
  scorer (QualityScorer.java) is deterministic and does NOT make an LLM call.
- Two guardrails on the same agent: ReplyGuardrail (before-agent-response) and
  BookingGuardrail (before-tool-call). Both registered via the agent's configuration block.
- The before-tool-call guardrail is wired via the agent's tool-call-guardrail-configuration
  mechanism, not as an ad-hoc check outside the agent loop.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
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
