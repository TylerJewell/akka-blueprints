# SPEC — steered-renewal-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** SteeredRenewalAgent.
**One-line pitch:** A patron submits a book-renewal request; one AI agent reads the patron record and loan state and returns a structured renewal decision — APPROVED / DENIED / EXTENDED with a reason — while a `before-agent-response` guardrail enforces the library's renewal policy before any decision leaves the agent loop.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the cx-support domain, with **steering** as the governing mechanism. One `RenewalAgent` (AutonomousAgent) carries the entire decision; the surrounding components prepare its input and enforce policy on its output before anything is persisted.

The governance mechanism is a `before-agent-response` guardrail (`PolicyEnforcer`) that validates every candidate `RenewalDecision` against a set of hard policy rules:

- A patron with outstanding fines above the configured threshold cannot receive an APPROVED or EXTENDED decision.
- An item at or above the maximum renewal count cannot be renewed.
- A decision's `newDueDate` must be after `Instant.now()` and no further than `maxRenewalWindowDays` in the future.
- The `outcome` enum must be one of `{APPROVED, DENIED, EXTENDED}`.

A malformed or policy-violating candidate triggers a retry inside the same task, so the patron never sees an invalid decision. This is **steering**: the guardrail does not merely log violations — it prevents them from propagating.

## 3. User-facing flows

The user opens the App UI tab.

1. The user picks a **patron** from a dropdown (three seeded patrons — one with no fines, one with fines below the threshold, one with fines above the threshold) or enters a custom patron ID.
2. The user picks a **loan** from the matching dropdown (each seeded patron has two loans — one eligible, one at max renewals).
3. The user clicks **Request renewal**. The UI POSTs to `/api/renewals` and receives a `renewalId`.
4. The card appears in the live list in `REQUESTED` state. Within ~1 s, it transitions to `ENRICHED` as the workflow loads the patron record and loan detail.
5. Within ~10–30 s, the `decideStep` completes. The card transitions to `DECIDING` then `DECISION_RECORDED`. The decision appears: an outcome badge (APPROVED / DENIED / EXTENDED), a reason sentence, and (if approved or extended) a `newDueDate`.
6. Within ~1 s of `DECISION_RECORDED`, the `notifyStep` records a `NotificationSent` event. The card reaches `COMPLETED`.
7. The user can submit another renewal; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RenewalEndpoint` | `HttpEndpoint` | `/api/renewals/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `LoanEntity`, `RenewalView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `LoanEntity` | `EventSourcedEntity` | Per-loan lifecycle: requested → enriched → deciding → decision → completed. Source of truth. | `RenewalEndpoint`, `RenewalWorkflow` | `RenewalView` |
| `RenewalWorkflow` | `Workflow` | One workflow per renewalId. Steps: `enrichStep` → `decideStep` → `notifyStep`. | started by `RenewalEndpoint` after `LoanEntity.requestRenewal` | `RenewalAgent`, `LoanEntity` |
| `RenewalAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the patron record and loan detail as task instructions; returns `RenewalDecision`. Governed by `PolicyEnforcer` guardrail. | invoked by `RenewalWorkflow` | returns decision |
| `PolicyEnforcer` | guardrail (`before-agent-response`) | Validates every candidate `RenewalDecision` against hard policy rules before it leaves the agent loop. | wired on `RenewalAgent` | agent loop (pass or retry) |
| `RenewalView` | `View` | Read model: one row per renewal for the UI. | `LoanEntity` events | `RenewalEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record PatronRecord(
    String patronId,
    String displayName,
    String patronTier,         // "STANDARD" | "PREMIUM" | "STAFF"
    double outstandingFinesCents,
    int lifetimeRenewals
) {}

record LoanDetail(
    String loanId,
    String itemBarcode,
    String itemTitle,
    String itemType,           // "BOOK" | "PERIODICAL" | "MEDIA"
    Instant originalDueDate,
    int priorRenewalCount,
    int maxRenewalsAllowed
) {}

record RenewalRequest(
    String renewalId,
    String patronId,
    String loanId,
    Instant requestedAt
) {}

record RenewalDecision(
    Outcome outcome,
    String reason,
    Optional<Instant> newDueDate,
    Instant decidedAt
) {}
enum Outcome { APPROVED, DENIED, EXTENDED }

record NotificationRecord(
    String channel,            // "in-app" | "email" | "sms"
    String message,
    Instant sentAt
) {}

record Renewal(
    String renewalId,
    Optional<RenewalRequest> request,
    Optional<PatronRecord> patron,
    Optional<LoanDetail> loan,
    Optional<RenewalDecision> decision,
    Optional<NotificationRecord> notification,
    RenewalStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RenewalStatus {
    REQUESTED, ENRICHED, DECIDING, DECISION_RECORDED, COMPLETED, FAILED
}
```

Events on `LoanEntity`: `RenewalRequested`, `LoanEnriched`, `DecisionStarted`, `DecisionRecorded`, `NotificationSent`, `RenewalFailed`.

Every nullable lifecycle field on the `Renewal` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/renewals` — body `{ patronId, loanId }` → `{ renewalId }`.
- `GET /api/renewals` — list all renewals, newest-first.
- `GET /api/renewals/{id}` — one renewal.
- `GET /api/renewals/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Library Book Renewal (Steering)</title>`.

The App UI tab is a two-column layout: a left rail with the Renewal Request form and the live list of submitted renewals (status pill + outcome badge + age) and a right pane with the selected renewal's detail — patron summary, loan detail, decision outcome, reason, new due date (if any), and notification record.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail**: runs on every turn of `RenewalAgent`. `PolicyEnforcer` validates the candidate `RenewalDecision` against four hard policy rules: (1) a patron above the fine threshold cannot receive APPROVED or EXTENDED; (2) an item at `priorRenewalCount >= maxRenewalsAllowed` cannot receive APPROVED or EXTENDED; (3) `newDueDate`, when present, must be after `now` and within `maxRenewalWindowDays`; (4) `outcome` must be in `{APPROVED, DENIED, EXTENDED}`. On any failure, returns a structured `policy-violation` error to the agent loop so the task retries within its iteration budget. Only a decision that passes all four checks reaches `decideStep`'s success path and is written to `LoanEntity`.

## 9. Agent prompts

- `RenewalAgent` → `prompts/renewal-agent.md`. The single decision-making LLM. System prompt instructs it to read the patron record and loan detail provided in the task, apply the library's renewal policy, and return a `RenewalDecision` with an outcome, a reason sentence, and (for APPROVED/EXTENDED) a `newDueDate`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Patron with no outstanding fines and an eligible loan submits a renewal request; within 30 s the card reaches COMPLETED with outcome APPROVED and a `newDueDate` 14 days out.
2. **J2** — The agent returns an APPROVED decision for a patron above the fine threshold; the `before-agent-response` guardrail rejects it; the second iteration returns DENIED with a reason citing the fine balance; the UI never shows the rejected candidate.
3. **J3** — A loan at `priorRenewalCount == maxRenewalsAllowed` is submitted; the agent returns DENIED with reason "Maximum renewal count reached"; the card reaches COMPLETED.
4. **J4** — The live SSE feed delivers each state transition to the UI without a page reload; multiple browser tabs connected simultaneously all update in real time.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named steered-renewal-agent demonstrating the single-agent × cx-support cell
with a before-agent-response steering guardrail. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact single-agent-cx-support-steered-renewal-agent.
Java package io.akka.samples.librarybookrenewalsteering. Akka 3.6.0. HTTP port 9354.

Components to wire (exactly):

- 1 AutonomousAgent RenewalAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/renewal-agent.md>) and
  .capability(TaskAcceptance.of(RENEW_LOAN).maxIterationsPerTask(3)). The task receives
  the patron record and loan detail formatted as task instructions (NOT as an attachment;
  the data is structured JSON, not a document body). Output:
  RenewalDecision{outcome: Outcome (APPROVED/DENIED/EXTENDED), reason: String,
  newDueDate: Optional<Instant>, decidedAt: Instant}. The agent is configured with a
  before-agent-response guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries the response
  within its 3-iteration budget.

- 1 Workflow RenewalWorkflow per renewalId with three steps:
  * enrichStep — loads PatronRecord and LoanDetail from in-process lookup tables
    (seeded from sample-events/seed-loans.jsonl); calls LoanEntity.attachEnrichedData.
    WorkflowSettings.stepTimeout 10s.
  * decideStep — emits DecisionStarted, then calls componentClient.forAutonomousAgent(
    RenewalAgent.class, "renewal-agent-" + renewalId).runSingleTask(
      TaskDef.instructions(formatRenewalContext(patron, loan))
    ) — returns a taskId, then forTask(taskId).result(RENEW_LOAN) to fetch the decision.
    On success calls LoanEntity.recordDecision(decision). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(2).failoverTo(RenewalWorkflow::error).
  * notifyStep — builds a NotificationRecord (channel "in-app", message summarising the
    outcome), calls LoanEntity.recordNotification(notification). Marks the loan COMPLETED.
    WorkflowSettings.stepTimeout 5s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity LoanEntity (one per renewalId). State
  Renewal{renewalId: String, request: Optional<RenewalRequest>,
  patron: Optional<PatronRecord>, loan: Optional<LoanDetail>,
  decision: Optional<RenewalDecision>, notification: Optional<NotificationRecord>,
  status: RenewalStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  RenewalStatus enum: REQUESTED, ENRICHED, DECIDING, DECISION_RECORDED, COMPLETED, FAILED.
  Events: RenewalRequested{request}, LoanEnriched{patron, loan}, DecisionStarted{},
  DecisionRecorded{decision}, NotificationSent{notification}, RenewalFailed{reason}.
  Commands: requestRenewal, attachEnrichedData, markDeciding, recordDecision,
  recordNotification, fail, getRenewal. emptyState() returns Renewal.initial("") with no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 View RenewalView with row type RenewalRow (mirrors Renewal; all fields included since
  there is no raw-document equivalent to exclude). Table updater consumes LoanEntity events.
  ONE query getAllRenewals: SELECT * AS renewals FROM renewal_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * RenewalEndpoint at /api with POST /renewals (body {patronId, loanId}; mints renewalId;
    calls LoanEntity.requestRenewal then starts RenewalWorkflow; returns {renewalId}),
    GET /renewals (list from getAllRenewals, sorted newest-first), GET /renewals/{id}
    (one row), GET /renewals/sse (Server-Sent Events forwarded from the view's
    stream-updates), and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- RenewalTasks.java declaring one Task<R> constant: RENEW_LOAN = Task.name("Renew loan")
  .description("Read the patron record and loan detail, apply renewal policy, and return a
  RenewalDecision").resultConformsTo(RenewalDecision.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records PatronRecord, LoanDetail, RenewalRequest, RenewalDecision, Outcome,
  NotificationRecord, Renewal, RenewalStatus.

- PolicyEnforcer.java implementing the before-agent-response hook. Reads the candidate
  RenewalDecision from the LLM response, runs the four policy checks listed in
  eval-matrix.yaml G1, and either passes the response through or returns
  Guardrail.reject(<structured-error>) naming the violated rule. The four checks are:
  (1) patron.outstandingFinesCents > FINE_THRESHOLD_CENTS → outcome must be DENIED.
  (2) loan.priorRenewalCount >= loan.maxRenewalsAllowed → outcome must be DENIED.
  (3) decision.newDueDate, when present, must be after Instant.now() and no more than
      MAX_RENEWAL_WINDOW_DAYS days in the future.
  (4) outcome must be one of {APPROVED, DENIED, EXTENDED}.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9354 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The RenewalAgent.definition() binds
  the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-loans.jsonl with 3 seeded patron+loan scenarios:
  (a) STANDARD patron, zero fines, loan at priorRenewalCount=0 (maxRenewalsAllowed=3) →
      eligible for APPROVED.
  (b) PREMIUM patron, fines below threshold (500 cents), loan at priorRenewalCount=2 →
      eligible for EXTENDED.
  (c) STANDARD patron, fines above threshold (2000 cents, threshold=1000 cents), loan at
      priorRenewalCount=0 → must be DENIED by policy.
  Each patron has two loans so the UI dropdown shows a choice between an eligible and a
  non-eligible item per patron.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with decisions.authority_level = enforced (the agent's
  decision is what the system acts on — not merely advisory), oversight.human_in_loop = false
  (renewal is automated), failure.failure_modes including "policy-violation-approved",
  "max-renewal-exceeded", "stale-patron-data", "fine-threshold-bypass"; deployer fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/renewal-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Library Book Renewal (Steering)",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  Renewal Request form + live list of renewal cards; right = selected-renewal detail with
  patron summary, loan detail, decision outcome badge, reason, new due date, notification
  record). Browser title exactly: <title>Akka Sample: Library Book Renewal (Steering)</title>.
  No subtitle on the Overview tab.

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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(renewalId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    renew-loan.json — 8 RenewalDecision entries covering the three Outcome values.
      Entries for APPROVED and EXTENDED include a newDueDate 14 days from now; DENIED
      entries have newDueDate absent. Reason strings are natural-language sentences.
      Plus 2 deliberately POLICY-VIOLATING entries (one APPROVED for a patron above the
      fine threshold; one EXTENDED for a loan at maxRenewalsAllowed) — the guardrail
      blocks both, exercising the retry path. The mock should select a violating entry on
      the FIRST iteration of every 3rd renewal (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(renewalId) helper makes per-renewal selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. RenewalAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion RenewalTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (decideStep
  60s, enrichStep 10s, notifyStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Renewal row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: RenewalTasks.java with RENEW_LOAN = Task.name(...).description(...)
  .resultConformsTo(RenewalDecision.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9354 declared explicitly in application.conf's
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (RenewalAgent). The
  notification step is a deterministic in-process record write — NOT an LLM call — keeping
  the pattern's "one agent" promise honest.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent returns. Lesson 1's
  AutonomousAgent contract is the authoritative reference for how the hook is registered.
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
