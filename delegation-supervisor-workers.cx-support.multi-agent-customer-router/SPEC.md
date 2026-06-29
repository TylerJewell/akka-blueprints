# SPEC — multi-agent-customer-router

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Multi-Agent Customer Router.
**One-line pitch:** Submit a support ticket; a router classifies it and delegates to the right specialist agent — Billing, Technical, or Account Management — with every routing decision audited before handoff.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow runs a before-invocation guardrail on the router's decision, delegates to one of three specialist AutonomousAgents, and persists the resolved outcome on an EventSourcedEntity. The blueprint also demonstrates an **eval-event** that scores whether the chosen specialist actually closed the ticket, making routing quality observable over time.

## 3. User-facing flows

The user opens the App UI tab and submits a ticket via the form.

1. The system creates a `Ticket` record in `OPEN` and starts a `TicketWorkflow`.
2. `RouterAgent` reads the ticket subject and body, emitting a `RoutingDecision { category, rationale }`.
3. A before-invocation guardrail audits the routing decision. If the decision fails validation (e.g., low-confidence classification), the ticket moves to `REVIEW_REQUIRED` and no specialist is called.
4. If the guardrail passes, the workflow delegates to the appropriate specialist: `BillingAgent`, `TechnicalAgent`, or `AccountAgent`.
5. The specialist returns a `Resolution { answer, resolvedAt, closedBy }`. The ticket moves to `RESOLVED`.
6. If the specialist times out after 90 seconds, the ticket moves to `ESCALATED` with a `failureReason`.

A `TicketSimulator` (TimedAction) drips a sample ticket every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RouterAgent` | `AutonomousAgent` | Classifies an incoming ticket into one of three categories; produces a `RoutingDecision`. | `TicketWorkflow` | returns typed result to workflow |
| `BillingAgent` | `AutonomousAgent` | Resolves billing-related enquiries. | `TicketWorkflow` | — |
| `TechnicalAgent` | `AutonomousAgent` | Resolves technical support queries. | `TicketWorkflow` | — |
| `AccountAgent` | `AutonomousAgent` | Handles account-management requests (upgrades, cancellations, profile changes). | `TicketWorkflow` | — |
| `TicketWorkflow` | `Workflow` | Runs routing guardrail, delegates to the chosen specialist, persists the outcome. | `TicketEndpoint`, `TicketQueueConsumer` | `TicketEntity` |
| `TicketEntity` | `EventSourcedEntity` | Holds the ticket's lifecycle (open → routing → in-progress → resolved / escalated / review-required). | `TicketWorkflow` | `TicketView` |
| `TicketQueueEntity` | `EventSourcedEntity` | Logs each submitted ticket for replay/audit. | `TicketEndpoint`, `TicketSimulator` | `TicketQueueConsumer` |
| `TicketView` | `View` | List-of-tickets read model. | `TicketEntity` events | `TicketEndpoint` |
| `TicketQueueConsumer` | `Consumer` | Listens to `TicketQueueEntity` events and starts a workflow per submission. | `TicketQueueEntity` events | `TicketWorkflow` |
| `TicketSimulator` | `TimedAction` | Drips a sample ticket every 60 s. | scheduler | `TicketQueueEntity` |
| `EvalSampler` | `TimedAction` | Samples one resolved ticket every 5 minutes for eval scoring; emits a `ResolutionEvalScored` event. | scheduler | `TicketEntity` |
| `TicketEndpoint` | `HttpEndpoint` | `/api/tickets/*` — submit, get, list, SSE. | — | `TicketView`, `TicketQueueEntity`, `TicketEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record TicketRequest(String subject, String body, String submittedBy) {}

record RoutingDecision(TicketCategory category, String rationale, double confidence) {}

enum TicketCategory { BILLING, TECHNICAL, ACCOUNT_MANAGEMENT }

record Resolution(String answer, Instant resolvedAt, String closedBy) {}

record Ticket(
    String ticketId,
    String subject,
    String body,
    String submittedBy,
    TicketStatus status,
    Optional<TicketCategory> category,
    Optional<String> routingRationale,
    Optional<Resolution> resolution,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum TicketStatus { OPEN, ROUTING, IN_PROGRESS, RESOLVED, ESCALATED, REVIEW_REQUIRED }
```

### Events (on `TicketEntity`)

`TicketOpened`, `TicketRouted`, `TicketAccepted`, `TicketResolved`, `TicketEscalated`, `TicketFlaggedForReview`, `ResolutionEvalScored`.

### Events (on `TicketQueueEntity`)

`TicketSubmitted { ticketId, subject, body, submittedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/tickets` — body `{ subject, body, submittedBy }` → `{ ticketId }`. Starts a workflow.
- `GET /api/tickets` — list all tickets. Optional `?status=OPEN|ROUTING|IN_PROGRESS|RESOLVED|ESCALATED|REVIEW_REQUIRED`.
- `GET /api/tickets/{id}` — one ticket.
- `GET /api/tickets/sse` — server-sent events stream of every ticket change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Multi-Agent Customer Router"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a ticket, live list of tickets with status pills, expand-row to see routing decision + specialist answer + eval score.

Browser title: `<title>Akka Sample: Multi-Agent Customer Router</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — routing guardrail** (`before-agent-invocation` on `RouterAgent`): audits the routing decision before the specialist is called. A low-confidence or contradictory classification moves the ticket to `REVIEW_REQUIRED` without invoking any specialist.
- **E1 — eval-event sampler** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one resolved ticket every 5 minutes and emits a `ResolutionEvalScored` event with a 1–5 score and a short rationale.

## 9. Agent prompts

- `RouterAgent` → `prompts/router.md`. Classifies the ticket; returns `RoutingDecision`.
- `BillingAgent` → `prompts/billing.md`. Handles billing queries; returns `Resolution`.
- `TechnicalAgent` → `prompts/technical.md`. Handles technical queries; returns `Resolution`.
- `AccountAgent` → `prompts/account.md`. Handles account-management requests; returns `Resolution`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a billing ticket; ticket progresses OPEN → ROUTING → IN_PROGRESS → RESOLVED within 60 s; UI reflects each transition via SSE; `closedBy` shows "BillingAgent".
2. **J2** — Submit a deliberately ambiguous ticket (test fixture, confidence < 0.5); guardrail intercepts the routing decision; ticket enters REVIEW_REQUIRED; no specialist is called.
3. **J3** — Inject a specialist timeout (set specialist timeout to 1 s); ticket enters ESCALATED with a `failureReason`.
4. **J4** — Wait after a successful resolution; the ticket row shows an eval score (1–5) from `EvalSampler`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named multi-agent-customer-router demonstrating the
delegation-supervisor-workers × cx-support cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-cx-support-multi-agent-customer-router.
Java package io.akka.samples.multiagentcustomerrouter. Akka 3.6.0. HTTP port 9538.

Components to wire (exactly):
- 4 AutonomousAgents:
  * RouterAgent — definition() with capability(TaskAcceptance.of(ROUTE).maxIterationsPerTask(2)).
    System prompt loaded from prompts/router.md. Returns
    RoutingDecision{category: TicketCategory, rationale: String, confidence: double}.
  * BillingAgent — capability(TaskAcceptance.of(RESOLVE_BILLING).maxIterationsPerTask(3)).
    System prompt from prompts/billing.md. Returns Resolution{answer, resolvedAt, closedBy}.
  * TechnicalAgent — capability(TaskAcceptance.of(RESOLVE_TECHNICAL).maxIterationsPerTask(3)).
    System prompt from prompts/technical.md. Returns Resolution{answer, resolvedAt, closedBy}.
  * AccountAgent — capability(TaskAcceptance.of(RESOLVE_ACCOUNT).maxIterationsPerTask(3)).
    System prompt from prompts/account.md. Returns Resolution{answer, resolvedAt, closedBy}.

- 1 Workflow TicketWorkflow with steps:
  routeStep -> guardrailStep -> delegateStep -> resolveStep -> emitStep.
  routeStep calls forAutonomousAgent(RouterAgent.class, ROUTE); give routeStep a 30s
  stepTimeout. guardrailStep runs a deterministic confidence-threshold check (threshold 0.5)
  plus an LLM judge; on failure, end with TicketFlaggedForReview and status REVIEW_REQUIRED
  — do not proceed to delegateStep. delegateStep branches by RoutingDecision.category:
    BILLING -> forAutonomousAgent(BillingAgent.class, RESOLVE_BILLING)
    TECHNICAL -> forAutonomousAgent(TechnicalAgent.class, RESOLVE_TECHNICAL)
    ACCOUNT_MANAGEMENT -> forAutonomousAgent(AccountAgent.class, RESOLVE_ACCOUNT)
  Give delegateStep a 90s stepTimeout. On timeout, route to escalateStep which calls
  TicketEntity.escalate and ends with TicketEscalated. WorkflowSettings is nested inside
  Workflow — no import.

- 1 EventSourcedEntity TicketEntity holding state Ticket{ticketId, subject, body,
  submittedBy, TicketStatus, Optional<TicketCategory> category,
  Optional<String> routingRationale, Optional<Resolution> resolution,
  Optional<String> failureReason, Optional<Integer> evalScore,
  Optional<String> evalRationale, Instant createdAt, Optional<Instant> finishedAt}.
  TicketStatus enum: OPEN, ROUTING, IN_PROGRESS, RESOLVED, ESCALATED, REVIEW_REQUIRED.
  Events: TicketOpened, TicketRouted, TicketAccepted, TicketResolved, TicketEscalated,
  TicketFlaggedForReview, ResolutionEvalScored.
  Commands: openTicket, routeTicket, acceptTicket, resolveTicket, escalateTicket,
  flagForReview, recordEval, getTicket.
  emptyState() returns Ticket.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity TicketQueueEntity with command submitTicket(subject, body,
  submittedBy) emitting TicketSubmitted{ticketId, subject, body, submittedBy, submittedAt}.

- 1 View TicketView with row type TicketRow (mirrors Ticket minus large nested payloads;
  every nullable field is Optional<T>). Table updater consumes TicketEntity events. ONE
  query getAllTickets SELECT * AS tickets FROM ticket_view. No WHERE status filter (Akka
  cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer TicketQueueConsumer subscribed to TicketQueueEntity events; on TicketSubmitted
  starts a TicketWorkflow with the ticketId as the workflow id.

- 2 TimedActions:
  * TicketSimulator — every 60s, reads next line from
    src/main/resources/sample-events/sample-tickets.jsonl and calls
    TicketQueueEntity.submitTicket.
  * EvalSampler — every 5 minutes, queries TicketView.getAllTickets, picks the oldest
    RESOLVED ticket without an evalScore, runs a 1–5 rubric judge over the resolution
    content, then calls TicketEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * TicketEndpoint at /api with POST /tickets, GET /tickets, GET /tickets/{id},
    GET /tickets/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- TicketTasks.java declaring four Task<R> constants: ROUTE (RoutingDecision),
  RESOLVE_BILLING (Resolution), RESOLVE_TECHNICAL (Resolution), RESOLVE_ACCOUNT (Resolution).
- Domain records TicketRequest, RoutingDecision, Resolution, Ticket.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9538 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/sample-tickets.jsonl with 8 canned ticket lines covering
  billing, technical, and account-management scenarios.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 routing guardrail
  before-agent-invocation, E1 eval-event on-decision-eval) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function =
  customer-support-routing, decisions.authority_level = recommend-only,
  data.data_classes.pii = true (ticket subjects contain customer identifiers),
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/router.md, prompts/billing.md, prompts/technical.md, prompts/account.md loaded
  at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Multi-Agent Customer Router", one-line
  pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no
  ui/ folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets), Risk Survey (7
  sub-tabs from governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/
  Control/Mechanism/Implementation/Source table with click-to-expand rows), App UI (form +
  live list with status pills). Browser title exactly:
  <title>Akka Sample: Multi-Agent Customer Router</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material. Akka
  records only the REFERENCE (env-var name, file path, secrets URI); the
  value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (router.json,
  billing.json, technical.json, account.json), picks one entry pseudo-randomly
  per call, and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    router.json — 8 RoutingDecision entries: 3 BILLING, 3 TECHNICAL,
      2 ACCOUNT_MANAGEMENT, all with confidence >= 0.7 and a one-sentence
      rationale.
    billing.json — 4–6 Resolution entries with realistic billing answers
      (e.g. "Your invoice #INV-2248 was paid on 15 Jun; the balance is $0."),
      resolvedAt timestamps, closedBy = "BillingAgent".
    technical.json — 4–6 Resolution entries with step-by-step technical
      answers (e.g. "Clear the app cache under Settings → Storage, then
      restart."), closedBy = "TechnicalAgent".
    account.json — 4–6 Resolution entries for account scenarios (e.g.
      "Your plan has been upgraded to Pro effective today."),
      closedBy = "AccountAgent".
- A MockModelProvider.seedFor(ticketId) helper makes the selection
  deterministic per ticket id so the same ticket produces the same output
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: routeStep gets 30s stepTimeout; delegateStep gets 90s stepTimeout. The 5s
  default fails every LLM call. WorkflowSettings nested inside Workflow — no import.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion TicketTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9538 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme
  variables (state-diagram label colour, edge-label foreignObject overflow:visible,
  transitionLabelColor #cccccc).
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index;
  delete removed panels from the DOM, do not display:none them.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK in
  narrative, T1/T2/T3/T4, deferred, marketing tone.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing options from Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
