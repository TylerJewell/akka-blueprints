# SPEC — support-next-steps

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** SupportNextSteps.
**One-line pitch:** A user submits a support ticket and a resolution library; one AI agent reads the ticket (passed as a task attachment, never as inline prompt text) and returns a structured `RecommendationSet` — a ranked list of next steps grounded in past resolutions, each with a confidence level and a citation to the resolution that backs it.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the cx-support domain. One `ResolutionAdvisorAgent` (AutonomousAgent) carries the entire decision; the surrounding components prepare its input and audit its output. Two governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw ticket submission and the agent call — so the model never sees customer identifiers, email addresses, account numbers, or phone numbers that appear in ticket bodies.
- An **on-decision-eval** runs immediately after each `RecommendationRecorded` event, scoring the recommendation set for grounding quality: are the suggested next steps actually backed by citations from the resolution library, or does the agent free-form advice that has no historical precedent?

The blueprint shows that the single-agent pattern does not mean "ungoverned" — two independent checks sit on either side of the one decision-making LLM call, and the output is scored before it surfaces in the UI.

## 3. User-facing flows

The user opens the App UI tab.

1. The user pastes a ticket body into the **Ticket** textarea (or picks one of three seeded examples — a billing dispute, a login-flow error, a data-export failure).
2. The user fills in a **subject** line and selects a **product area** from a dropdown (Billing / Authentication / Data Export).
3. The user clicks **Submit ticket**. The UI POSTs to `/api/tickets` and receives a `ticketId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `SANITIZED` — the redacted ticket body is visible in the card detail, with a small list of PII categories the sanitizer found.
5. Within ~10–30 s, the workflow's `adviseStep` completes. The card transitions to `ADVISING` then `RECOMMENDATION_RECORDED`. The recommendation appears: a top-level confidence badge (HIGH / MEDIUM / LOW), a short rationale paragraph, and a ranked step list — each step carrying a `stepNumber`, a `description`, an `actionType`, a `confidence` level, and a `resolutionRef` citing the past case.
6. Within ~1 s of the recommendation, the `evalStep` finishes. The card shows an **eval score** chip (1–5) plus a one-line rationale describing whether the recommendations are grounded.
7. The user can submit another ticket; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TicketEndpoint` | `HttpEndpoint` | `/api/tickets/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `TicketEntity`, `TicketView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `TicketEntity` | `EventSourcedEntity` | Per-ticket lifecycle: submitted → sanitized → advising → recommendation → eval. Source of truth. | `TicketEndpoint`, `TicketSanitizer`, `TicketWorkflow` | `TicketView` |
| `TicketSanitizer` | `Consumer` | Subscribes to `TicketSubmitted` events; redacts PII; calls `TicketEntity.attachSanitized`. | `TicketEntity` events | `TicketEntity` |
| `TicketWorkflow` | `Workflow` | One workflow per ticket. Steps: `awaitSanitizedStep` → `adviseStep` → `evalStep`. | started by `TicketSanitizer` once sanitized event lands | `ResolutionAdvisorAgent`, `TicketEntity` |
| `ResolutionAdvisorAgent` | `AutonomousAgent` | The one decision-making LLM. Receives ticket context in the task definition and the sanitized ticket plus the resolution library as task attachments; returns `RecommendationSet`. | invoked by `TicketWorkflow` | returns recommendation |
| `TicketView` | `View` | Read model: one row per ticket for the UI. | `TicketEntity` events | `TicketEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ResolutionRef(String caseId, String summary, String resolutionDate) {}

record TicketRequest(
    String ticketId,
    String subject,
    String ticketBody,
    String productArea,
    String submittedBy,
    Instant submittedAt
) {}

record SanitizedTicket(
    String redactedBody,
    List<String> piiCategoriesFound
) {}

record RecommendedStep(
    int stepNumber,
    String description,
    ActionType actionType,
    Confidence confidence,
    String resolutionRef   // caseId of backing past resolution, or empty if none
) {}
enum ActionType { VERIFY, ESCALATE, CONFIGURE, INFORM, INVESTIGATE }
enum Confidence { HIGH, MEDIUM, LOW }

record RecommendationSet(
    OverallConfidence overallConfidence,
    String rationale,
    List<RecommendedStep> steps,
    Instant decidedAt
) {}
enum OverallConfidence { HIGH, MEDIUM, LOW }

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record Ticket(
    String ticketId,
    Optional<TicketRequest> request,
    Optional<SanitizedTicket> sanitized,
    Optional<RecommendationSet> recommendation,
    Optional<EvalResult> eval,
    TicketStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum TicketStatus {
    SUBMITTED, SANITIZED, ADVISING, RECOMMENDATION_RECORDED, EVALUATED, FAILED
}
```

Events on `TicketEntity`: `TicketSubmitted`, `TicketSanitized`, `AdvisingStarted`, `RecommendationRecorded`, `EvaluationScored`, `TicketFailed`.

Every nullable lifecycle field on the `Ticket` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/tickets` — body `{ subject, ticketBody, productArea, submittedBy }` → `{ ticketId }`.
- `GET /api/tickets` — list all tickets, newest-first.
- `GET /api/tickets/{id}` — one ticket.
- `GET /api/tickets/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: SupportNextSteps</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted tickets (status pill + confidence badge + age) and a right pane with the selected ticket's detail — sanitized ticket body preview, recommendation rationale, ranked step list with confidence chips, and eval-score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `TicketSanitizer` Consumer): redacts emails, phone numbers, account numbers, customer names, postal addresses, and support-ticket-reference-like identifiers from the raw ticket body before any LLM call. Records which categories were found.
- **E1 — on-decision eval** (`eval-event`, `on-decision-eval`): runs immediately after `RecommendationRecorded` lands, as `evalStep` inside the workflow. A deterministic scorer (no LLM call — the eval is rule-based on purpose, so the same recommendation always scores the same) checks that each `RecommendedStep.resolutionRef` is non-empty, that recommendations use actionable language, and that confidence levels are not uniformly collapsed to a single value across all steps. Emits `EvaluationScored` with a 1–5 score and a one-line rationale.

## 9. Agent prompts

- `ResolutionAdvisorAgent` → `prompts/resolution-advisor.md`. The single decision-making LLM. System prompt instructs it to read the sanitized ticket body, match it against the resolution library attachment, and return a `RecommendationSet` with ranked steps, each citing the past resolution it draws from.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the billing-dispute seed ticket; within 30 s the recommendation appears with at least two ranked steps and an eval score chip.
2. **J2** — A recommendation set whose steps all carry empty `resolutionRef` strings receives an eval score of 1 with a clear rationale; the UI flags the card.
3. **J3** — A ticket containing `jane.doe@acme.com` and `account: 4829-XXXX` is submitted; the LLM call log shows only `[REDACTED-EMAIL]` and `[REDACTED-ACCOUNT]`; the entity's `request.ticketBody` retains the raw text for audit.
4. **J4** — The live list updates in real time over SSE; a client that connects after the ticket reaches `EVALUATED` receives the full row immediately on the first SSE event.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named support-next-steps demonstrating the single-agent × cx-support cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-cx-support-support-next-steps. Java package
io.akka.samples.supportticketnextsteps. Akka 3.6.0. HTTP port 9220.

Components to wire (exactly):

- 1 AutonomousAgent ResolutionAdvisorAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/resolution-advisor.md>) and
  .capability(TaskAcceptance.of(ADVISE_TICKET).maxIterationsPerTask(3)). The task receives
  ticket context as its instruction text and two attachments: (1) the sanitized ticket body
  as "ticket.txt" and (2) the resolution library as "resolution-library.txt". Neither is
  inlined as prompt text — both use TaskDef.attachment(name, contentBytes). Output:
  RecommendationSet{overallConfidence: OverallConfidence (HIGH/MEDIUM/LOW), rationale: String,
  steps: List<RecommendedStep>, decidedAt: Instant}. There is no before-agent-response
  guardrail in this blueprint — structural validity is checked by the on-decision evaluator
  downstream.

- 1 Workflow TicketWorkflow per ticketId with three steps:
  * awaitSanitizedStep — polls TicketEntity.getTicket every 1s; on ticket.sanitized().isPresent()
    advances to adviseStep. WorkflowSettings.stepTimeout 15s (sanitizer is in-process and fast).
  * adviseStep — emits AdvisingStarted, then calls componentClient.forAutonomousAgent(
    ResolutionAdvisorAgent.class, "advisor-" + ticketId).runSingleTask(
      TaskDef.instructions(formatTicketContext(ticket.request()))
        .attachment("ticket.txt", ticket.sanitized().get().redactedBody().getBytes())
        .attachment("resolution-library.txt", loadResolutionLibrary(ticket.request().productArea()))
    ) — returns a taskId, then forTask(taskId).result(ADVISE_TICKET) to fetch the recommendation.
    On success calls TicketEntity.recordRecommendation(recommendation). WorkflowSettings
    .stepTimeout 60s with defaultStepRecovery maxRetries(2).failoverTo(TicketWorkflow::error).
  * evalStep — runs a deterministic rule-based RecommendationScorer (NOT an LLM call) over the
    recorded recommendation: checks that every step has a non-empty resolutionRef, that
    descriptions are actionable, and that confidence levels are not uniformly collapsed.
    Emits EvaluationScored{score: 1-5, rationale: String}. WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity TicketEntity (one per ticketId). State Ticket{ticketId: String,
  request: Optional<TicketRequest>, sanitized: Optional<SanitizedTicket>,
  recommendation: Optional<RecommendationSet>, eval: Optional<EvalResult>, status: TicketStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. TicketStatus enum: SUBMITTED,
  SANITIZED, ADVISING, RECOMMENDATION_RECORDED, EVALUATED, FAILED. Events: TicketSubmitted{request},
  TicketSanitized{sanitized}, AdvisingStarted{}, RecommendationRecorded{recommendation},
  EvaluationScored{eval}, TicketFailed{reason}. Commands: submit, attachSanitized,
  markAdvising, recordRecommendation, recordEvaluation, fail, getTicket. emptyState() returns
  Ticket.initial("") with no commandContext() reference (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer TicketSanitizer subscribed to TicketEntity events; on TicketSubmitted runs
  a regex+heuristic redaction pipeline (emails, phone numbers, account-number-like tokens,
  postal addresses, person-name heuristic, customer-id-like patterns) over ticketBody, computes
  the list of categories found, builds SanitizedTicket, then calls
  TicketEntity.attachSanitized(sanitized). After attachSanitized lands, the same Consumer
  starts a TicketWorkflow with id = "ticket-" + ticketId.

- 1 View TicketView with row type TicketRow (mirrors Ticket minus request.ticketBody — the
  audit log keeps the raw; the view holds the sanitized form for the UI). Table updater
  consumes TicketEntity events. ONE query getAllTickets: SELECT * AS tickets FROM ticket_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * TicketEndpoint at /api with POST /tickets (body {subject, ticketBody, productArea,
    submittedBy}; mints ticketId; calls TicketEntity.submit; returns {ticketId}), GET /tickets
    (list from getAllTickets, sorted newest-first), GET /tickets/{id} (one row), GET
    /tickets/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- TicketTasks.java declaring one Task<R> constant: ADVISE_TICKET = Task.name("Advise ticket")
  .description("Read the attached sanitized ticket and resolution library, then produce a
  RecommendationSet").resultConformsTo(RecommendationSet.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records TicketRequest, SanitizedTicket, ResolutionRef, RecommendedStep, ActionType,
  Confidence, RecommendationSet, OverallConfidence, EvalResult, Ticket, TicketStatus.

- RecommendationScorer.java — pure deterministic logic (no LLM). Inputs: RecommendationSet.
  Outputs: EvalResult. Scoring rubric: (1) each step's resolutionRef non-empty (+1 point per
  covered step, max 2); (2) each step description begins with an actionable verb (+1); (3)
  confidence levels not uniformly identical across all steps (+1); (4) at least 2 steps
  present (+1). Scale 1–5 documented in Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9220 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The ResolutionAdvisorAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/resolution-library.jsonl with 3 seeded resolution sets:
  a 6-entry billing-dispute resolution library, a 5-entry authentication-error library, and
  a 4-entry data-export-failure library. Each entry has a caseId, a summary, a resolutionDate,
  and a resolution-steps field.

- src/main/resources/sample-events/seed-tickets.jsonl with 3 paired example tickets: a billing
  dispute (300 words, contains a plausible email and account number), a login-flow error
  (250 words, contains a phone number and name), and a data-export failure (200 words, contains
  an email and IP address). Each contains 2–3 PII strings so S1 has work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (S1, E1) matching the mechanisms in
  Section 8 of this SPEC.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = recommend-only
  (the agent's suggestion is advisory, not enforced), oversight.human_in_loop = true (a
  support agent reads the steps before acting), failure.failure_modes including
  "ungrounded-recommendation", "missed-resolution-match", "pii-leakage-via-llm",
  "confidence-miscalibration"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/resolution-advisor.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: SupportNextSteps", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of ticket cards; right = selected-ticket detail with sanitized body preview,
  recommendation rationale, ranked step list with confidence chips, and eval-score chip).
  Browser title exactly: <title>Akka Sample: SupportNextSteps</title>. No subtitle on the
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(ticketId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    advise-ticket.json — 6 RecommendationSet entries covering all three OverallConfidence
      values. Each entry has a rationale paragraph and a steps array with 2–4 RecommendedStep
      entries. Each step has a non-empty description with an actionable verb, an actionType
      from the enum, a confidence level, and a non-empty resolutionRef citing a caseId from
      the matching resolution library. Include 1 "ungrounded" entry where all steps have
      empty resolutionRef strings — this exercises the eval score-1 path (J2). The mock
      selects the ungrounded entry on every 4th ticket (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(ticketId) helper makes per-ticket selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ResolutionAdvisorAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion TicketTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (adviseStep
  60s, awaitSanitizedStep 15s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Ticket row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: TicketTasks.java with ADVISE_TICKET = Task.name(...).description(...)
  .resultConformsTo(RecommendationSet.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9220 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more. Any tab removed in an earlier
  iteration must be deleted from the HTML; display:none is not enough.
- The single-agent invariant: there is exactly ONE AutonomousAgent (ResolutionAdvisorAgent).
  The on-decision eval is rule-based (RecommendationScorer.java) and does NOT make an LLM
  call — keeping the pattern's "one agent" promise honest.
- The ticket body and the resolution library are passed as Task ATTACHMENTS, never inlined
  into the agent's instructions. Verify the generated adviseStep uses TaskDef.attachment(...)
  for both, not string interpolation into the instruction text.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block. Per
  Lesson 25, /akka:specify handles the key during generation.
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
