# SPEC — akka-cx-with-escalation

The natural-language brief `/akka:specify` reads to generate this system. The whole file — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** OpenAI Agents Customer Service with Escalation.
**One-line pitch:** A customer submits a message; `SupportAgent` replies and decides whether to resolve or escalate; if escalation is warranted the workflow signals a human agent via the API; on human acceptance `EscalationAgent` produces a handoff summary and the conversation transitions to `ESCALATED`.

## 2. What this blueprint demonstrates

The human-in-loop-gate coordination pattern in the cx-support domain: a durable workflow that holds multi-turn conversation state and can pause at an escalation gate where a human agent accepts the hand-off. The governance pattern combines an application-level escalation gate (the canonical HITL for this domain), a PII sanitizer that strips personal data from messages before they are stored, and a before-agent-response guardrail that checks every customer-facing reply for tone and safe-messaging compliance.

## 3. User-facing flows

1. A client POSTs a customer message to `/api/conversations`. The response returns `{ conversationId }`. The conversation appears in the UI in `ACTIVE` once `SupportAgent` finishes (typically 5–30 s), with the agent reply visible.
2. The workflow — or the customer, via the API — triggers escalation. The conversation moves to `ESCALATING`. A human agent POSTs to `/api/conversations/{conversationId}/accept-escalation`. The workflow resumes, `EscalationAgent` runs, and the handoff summary appears with status `ESCALATED`.
3. `SupportAgent` resolves the issue directly. The conversation transitions to terminal `RESOLVED` and no escalation occurs.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| SupportAgent | AutonomousAgent | Handles customer messages; returns `AgentReply{message, action}` | SupportWorkflow | ConversationEntity |
| EscalationAgent | AutonomousAgent | Drafts human handoff summary; returns `HandoffSummary{summary, priority}` | SupportWorkflow | ConversationEntity |
| SupportWorkflow | Workflow | Orchestrates receive → reply → await escalation decision → escalate | SupportEndpoint | SupportAgent, EscalationAgent, ConversationEntity |
| ConversationEntity | EventSourcedEntity | Holds conversation state and lifecycle events | SupportWorkflow, SupportEndpoint | ConversationsView |
| ConversationsView | View | CQRS read model of all conversations | ConversationEntity | SupportEndpoint |
| SupportEndpoint | HttpEndpoint | REST + SSE + metadata surface | UI client | SupportWorkflow, ConversationEntity, ConversationsView |
| AppEndpoint | HttpEndpoint | Serves the static UI | Browser | static-resources |

## 5. Data model

`Conversation` (ConversationEntity state and ConversationsView row): `id` (String), `customerMessage` (`Optional<String>`), `status` (ConversationStatus enum), and lifecycle fields all `Optional<T>`: `openedAt`, `agentReply`, `repliedAt`, `escalationReason`, `escalationRequestedAt`, `acceptedBy`, `acceptedAt`, `handoffSummary`, `handoffPriority`, `resolvedAt`, `resolveNote`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

`ConversationStatus` enum: `ACTIVE`, `ESCALATING`, `ESCALATED`, `RESOLVED`.

Events: `ConversationOpened`, `ReplyRecorded`, `EscalationRequested`, `EscalationAccepted`, `ConversationResolved`.

Domain records: `AgentReply(String message, String action)`, `EscalationRequest(String reason)`, `HandoffSummary(String summary, String priority)`.

Full detail: `reference/data-model.md`.

## 6. API contract

Inline surface (schemas in `reference/api-contract.md`):

```
POST /api/conversations                                -> { conversationId }
POST /api/conversations/{conversationId}/escalate      -> 200 | 404
POST /api/conversations/{conversationId}/accept-escalation -> 200 | 404
POST /api/conversations/{conversationId}/resolve       -> 200 | 404
GET  /api/conversations                               -> { conversations: [Conversation, ...] }
GET  /api/conversations/{conversationId}              -> Conversation
GET  /api/conversations/sse                           -> Server-Sent Events of Conversation
GET  /api/metadata/eval-matrix                        -> text/yaml
GET  /api/metadata/risk-survey                        -> text/yaml
GET  /api/metadata/readme                             -> text/markdown
GET  /                                                -> 302 /app/index.html
GET  /app/{*path}                                     -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (no `ui/` folder, no npm build). Browser title: `<title>Akka Sample: OpenAI Agents Customer Service with Escalation</title>`. Five tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI. Tab switching matches by `data-tab` / `data-panel` attribute, never NodeList index, and removed tabs are deleted from the DOM, not hidden (Lesson 26). The Architecture tab renders the PLAN mermaid diagrams with the theme variables and CSS overrides from Lesson 24. The App UI tab submits a customer message, lists conversations live via SSE, and shows Escalate/Resolve buttons on `ACTIVE` conversations and an Accept Escalation button on `ESCALATING` conversations. Full description: `reference/ui-mockup.md`.

## 8. Governance

Controls live in `eval-matrix.yaml`; the deployer survey in `risk-survey.yaml`. Mechanisms the generated system wires:

- **H1 — hitl · application.** `SupportWorkflow` pauses at the await-escalation-acceptance task; `/api/conversations/{id}/accept-escalation` resumes it.
- **S1 — sanitizer · pii.** A PII sanitizer runs over the inbound `customerMessage` before it is persisted on `ConversationEntity`, redacting recognized personal data patterns (email, phone, card numbers).
- **G1 — guardrail · before-agent-response.** A guardrail on `SupportAgent` checks tone, safe-messaging, and length before the `AgentReply` is recorded for the customer.

## 9. Agent prompts

- `SupportAgent` — handles customer messages, decides action. See `prompts/support-agent.md`.
- `EscalationAgent` — drafts a human handoff summary with priority. See `prompts/escalation-agent.md`.

## 10. Acceptance

Inlined journeys (full set in `reference/user-journeys.md`):

1. **Open a conversation.** POST a customer message; within ~30 s a conversation appears in `ACTIVE` with non-empty `agentReply`.
2. **Escalate and accept.** Escalate an `ACTIVE` conversation; it reaches `ESCALATING`; accept the escalation; it reaches `ESCALATED` with a non-null `handoffSummary`.
3. **Resolve directly.** Resolve an `ACTIVE` conversation; it moves to terminal `RESOLVED` with no `handoffSummary`.
4. **Escalation gate.** The escalation handoff step does not run if no human has accepted via the API.

---

## 11. Implementation directives

```
Create a sample named akka-cx-with-escalation demonstrating the human-in-loop-gate ×
cx-support cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact human-in-loop-gate-cx-support-akka-cx-with-escalation.
Java package io.akka.samples.openaiagentscustomerservicewithescalation.
Akka 3.6.0. HTTP port 9268.

Components to wire (exactly):
- 2 AutonomousAgents: SupportAgent (handles customer messages, decides resolve or
  escalate, returns a typed AgentReply{message,action}) and EscalationAgent (returns
  a typed HandoffSummary{summary,priority}). Each declares definition() returning an
  AgentDefinition with .instructions(...) loaded from prompts and
  .capability(TaskAcceptance.of(task).maxIterationsPerTask(3)). Both extend
  akka.javasdk.agent.autonomous.AutonomousAgent — never downgrade to Agent.
- 1 Workflow SupportWorkflow with steps: replyStep -> awaitEscalationDecisionStep
  -> acceptEscalationStep. replyStep calls forAutonomousAgent(SupportAgent.class,...)
  .runSingleTask(...) then forTask(taskId).result(...), writes recordReply on
  ConversationEntity. awaitEscalationDecisionStep polls ConversationEntity.getConversation;
  on ACTIVE it self-schedules a 5-second resume timer; on ESCALATING it transitions to
  acceptEscalationStep; on RESOLVED it ends. acceptEscalationStep polls until
  ConversationEntity.status == ESCALATING and acceptedBy is present, then calls
  EscalationAgent and writes recordHandoff. Override settings() with stepTimeout(60s)
  on replyStep and acceptEscalationStep; WorkflowSettings is nested in Workflow (no import).
- 1 EventSourcedEntity ConversationEntity holding a Conversation record with id,
  customerMessage (Optional<String>), ConversationStatus enum
  {ACTIVE,ESCALATING,ESCALATED,RESOLVED}, and Optional lifecycle fields (openedAt,
  agentReply, repliedAt, escalationReason, escalationRequestedAt, acceptedBy,
  acceptedAt, handoffSummary, handoffPriority, resolvedAt, resolveNote).
  Events: ConversationOpened, ReplyRecorded, EscalationRequested, EscalationAccepted,
  ConversationResolved. Commands: recordReply, requestEscalation, acceptEscalation,
  resolve, recordHandoff, getConversation. emptyState() returns
  Conversation.initial("") with no commandContext() reference (Lesson 3).
- 1 View ConversationsView with row type Conversation, table updater consuming
  ConversationEntity events. ONE query: getAllConversations SELECT * AS conversations
  FROM conversations_view. No WHERE status filter (Akka cannot auto-index enum columns,
  Lesson 2) — filter client-side in callers.
- 2 HttpEndpoints: SupportEndpoint at /api with conversations (starts a
  SupportWorkflow with a fresh UUID), escalate, accept-escalation, resolve,
  conversations list (filter client-side from getAllConversations), single conversation,
  SSE stream, and three /api/metadata/* endpoints serving the YAML/MD files from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html and
  /app/* -> static-resources/*.

Companion files:
- SupportTasks.java declaring two Task<R> constants: REPLY (resultConformsTo
  AgentReply) and HANDOFF (resultConformsTo HandoffSummary).
- AgentReply(String message, String action), EscalationRequest(String reason),
  HandoffSummary(String summary, String priority).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9268
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 3 controls (H1, S1, G1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data.types,
  capability.*, model.*, subjects.children; marking jurisdictions, declared
  frameworks, organization fields, incidents, and data.residency as
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root: elevator pitch, component inventory, matrix cell,
  integration descriptor, how to run, the 5 tabs, an ASCII architecture diagram,
  project layout, API contract, license. NO governance-mechanisms section. NO
  configuration section.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN imports for
  markdown and YAML libs are acceptable. Five tabs: Overview, Architecture, Risk
  Survey, Eval Matrix, App UI. Match the governance.html visual style
  (dark/yellow/Instrument Sans/dot-grid). Include the Lesson 24 mermaid CSS
  overrides and theme variables. Tab switching matches by data-tab/data-panel
  attribute (Lesson 26).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random outputs
  (SupportAgent -> AgentReply, EscalationAgent -> HandoffSummary; see
  src/main/resources/mock-responses/{support-agent,escalation-agent}.json with 4–6
  entries each). Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed to
  the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured reference if
  it does not resolve at runtime; never echoes any captured key.

Mock LLM provider — per-agent mock-response shapes (option a):
- support-agent.json: 4–6 entries, each { "message": "...", "action": "resolve|escalate" }.
- escalation-agent.json: 4–6 entries, each { "summary": "...", "priority": "low|medium|high|urgent" }.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: stepTimeout(60s) on every agent-calling workflow step.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: each AutonomousAgent has its companion Task<R> constant in
  SupportTasks.java.
- Lesson 8: verify model names are current before locking application.conf.
- Lesson 9: the run command is "/akka:build", never "mvn akka:run".
- Lesson 10: port 9268 declared in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never
  T1/T2/T3/T4 or "deferred".
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: index.html includes mermaid CSS overrides and theme variables
  (state-label colour, edge-label overflow:visible, transitionLabelColor #cccccc).
- Lesson 25: five-option key sourcing; never write a key value to disk.
- Lesson 26: tab switching matches by data-tab/data-panel attribute, never
  NodeList index; no hidden zombie panels in the DOM.
- Overview tab Try-it card is just "/akka:build" — no env-var export block.
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
