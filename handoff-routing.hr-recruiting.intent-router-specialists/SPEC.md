# SPEC — core-semantic-router

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Core Semantic Router.
**One-line pitch:** An intent-router agent classifies an inbound HR or Finance query and hands the resolution off to the matching domain specialist that owns it end-to-end, with PII redaction before any LLM call, a before-agent-invocation guardrail on the routing decision, and an inline eval scoring every classification.

## 2. What this blueprint demonstrates

The **handoff-routing** coordination pattern — one classifier agent decides *who* should own the query, then transfers the same task identity to a downstream specialist agent that produces the final answer. The downstream specialist is responsible for the whole response; the classifier does not narrate or summarise. Two governance mechanisms are layered on top:

- A **PII sanitizer** runs inside a Consumer between the raw query event and the LLM call. The router and the specialists never see raw employee IDs, SSNs, salary figures, or financial account references.
- A **before-agent-invocation guardrail** runs after the intent is classified but before the specialist is called. It checks that the routing target is a known specialist, that the confidence meets the minimum threshold, and that the caller's session is authorized for that domain. A failed check blocks the specialist from running entirely.
- An **on-decision eval** fires every time the router emits a routing decision. A `RoutingJudge` agent grades the decision against the sanitized payload on a 1–5 rubric. The score and rationale are written back to the query entity and surfaced in the UI.

The pattern is a textbook fan-out-of-one: the workflow branches on the classifier's domain, and only the chosen specialist is invoked. The other specialist sees no traffic for that query.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live query list. Every query displays its domain chip, status pill, routing score, and (if answered) the published response.
2. `QuerySimulator` (TimedAction) ticks every 30 s and inserts a new canned query from `sample-events/queries.jsonl` into `QueryQueue`.
3. For each new query: `PiiSanitizer` (Consumer) redacts the payload, registers a `QueryEntity`, and starts a `RouterWorkflow`.
4. The workflow calls `IntentRouterAgent`, gets a `RoutingDecision { domain, confidence, reason }`, and emits `IntentClassified` on the entity.
5. The workflow calls `RoutingGuardrail` with the `RoutingDecision` and caller context. If the verdict is `authorized`, the query moves to `AUTHORIZED`; if not, `QueryBlocked` is emitted (terminal `BLOCKED`) with the rejection reasons.
6. Branch on `domain`:
   - `HR` → workflow calls `HrSpecialist` with the `ANSWER` task and waits for the typed `QueryAnswer` result.
   - `FINANCE` → workflow calls `FinanceSpecialist` with the same `ANSWER` task.
   - `AMBIGUOUS` → workflow emits `QueryEscalated`; ends.
7. The specialist's `QueryAnswer` is published. `AnswerPublished` is emitted (terminal `ANSWERED`).
8. Independent of the workflow, `RoutingEvalScorer` (Consumer) listens for `IntentRouted` events, calls `RoutingJudge`, and writes `RoutingScored { score, rationale }` back to the query entity.
9. The user can click any query card and see the redacted payload, the routing reason, the routing score, the chosen specialist, and the published answer (or the guardrail violations when `BLOCKED`).

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QuerySimulator` | `TimedAction` | Drips simulated HR and Finance queries into `QueryQueue` every 30 s. | scheduler | `QueryQueue` |
| `QueryQueue` | `EventSourcedEntity` | Append-only audit log of every inbound query (`InboundQueryReceived`). | `QuerySimulator`, `RouterEndpoint` | `PiiSanitizer` |
| `PiiSanitizer` | `Consumer` | Subscribes to `QueryQueue` events; redacts PII; registers `QueryEntity`; starts a `RouterWorkflow`. | `QueryQueue` events | `QueryEntity`, `RouterWorkflow` |
| `IntentRouterAgent` | `Agent` (typed, not autonomous) | Classifies a `SanitizedQuery` into `HR` / `FINANCE` / `AMBIGUOUS` with confidence + reason. | invoked by `RouterWorkflow` | returns `RoutingDecision` |
| `RoutingGuardrail` | `Agent` (typed) | Before-agent-invocation guardrail: validates routing domain, confidence threshold, and caller authorization. Returns `GuardrailVerdict { authorized, rejections }`. | invoked by `RouterWorkflow` | returns `GuardrailVerdict` |
| `HrSpecialist` | `AutonomousAgent` | Owns the `ANSWER` task for HR queries. Returns typed `QueryAnswer`. | invoked by `RouterWorkflow` | returns `QueryAnswer` |
| `FinanceSpecialist` | `AutonomousAgent` | Owns the `ANSWER` task for Finance queries. Returns typed `QueryAnswer`. | invoked by `RouterWorkflow` | returns `QueryAnswer` |
| `RoutingJudge` | `Agent` (typed) | Grades a routing decision against the sanitized payload. Returns `RoutingScore { score 1–5, rationale }`. | invoked by `RoutingEvalScorer` | returns `RoutingScore` |
| `RouterWorkflow` | `Workflow` | Per-query orchestration: sanitize → classify → guardrail → route → answer → publish. | `PiiSanitizer` (start) | `QueryEntity` |
| `QueryEntity` | `EventSourcedEntity` | Per-query lifecycle. | `RouterWorkflow`, `RoutingEvalScorer` | `QueryView` |
| `QueryView` | `View` | Read-model row per query. | `QueryEntity` events | `RouterEndpoint` |
| `RoutingEvalScorer` | `Consumer` | Subscribes to `QueryEntity` events; on `IntentRouted` invokes `RoutingJudge` and writes `RoutingScored` back. | `QueryEntity` events | `QueryEntity` |
| `RouterEndpoint` | `HttpEndpoint` | `/api/queries/*` — list, get, manual submit, SSE; `/api/metadata/*`. | — | `QueryView`, `QueryEntity`, `QueryQueue` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and static resources. | — | static resources |

## 5. Data model

```java
record IncomingQuery(
    String queryId,
    String requesterId,
    String channel,           // "portal" | "slack" | "email"
    String subject,
    String body,
    Instant receivedAt
) {}

record SanitizedQuery(
    String redactedSubject,
    String redactedBody,
    List<String> piiCategoriesFound
) {}

enum QueryDomain { HR, FINANCE, AMBIGUOUS }

record RoutingDecision(
    QueryDomain domain,
    String confidence,        // "high" | "medium" | "low"
    String reason             // one short sentence
) {}

record GuardrailVerdict(
    boolean authorized,
    List<String> rejections,  // empty when authorized
    String rubricVersion
) {}

enum AnswerAction { POLICY_CITED, PROCESS_EXPLAINED, ESCALATED, REFERRED }

record QueryAnswer(
    String answerBody,
    AnswerAction action,
    String specialistTag,     // "hr" | "finance"
    Instant answeredAt
) {}

record RoutingScore(
    int score,                // 1..5
    String rationale,
    Instant scoredAt
) {}

record Query(
    String queryId,
    IncomingQuery incoming,
    Optional<SanitizedQuery> sanitized,
    Optional<RoutingDecision> routing,
    Optional<GuardrailVerdict> guardrail,
    Optional<QueryAnswer> answer,
    Optional<RoutingScore> routingScore,
    Optional<String> escalationReason,
    QueryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus {
    RECEIVED,
    SANITIZED,
    CLASSIFIED,
    AUTHORIZED,
    ROUTED_HR,
    ROUTED_FINANCE,
    ANSWERED,
    BLOCKED,
    ESCALATED
}
```

Events on `QueryEntity`: `QueryRegistered`, `QuerySanitized`, `IntentClassified`, `IntentRouted`, `GuardrailVerdictAttached`, `QueryAnswered`, `QueryBlocked`, `QueryEscalated`, `RoutingScored`.

Events on `QueryQueue`: `InboundQueryReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/queries` — list all queries (newest-first), optional `?domain=HR|FINANCE|AMBIGUOUS&status=…` filtered client-side.
- `GET /api/queries/{id}` — one query.
- `POST /api/queries` — manually submit a query (body `IncomingQuery` minus `queryId` and `receivedAt`); server assigns both.
- `GET /api/queries/sse` — Server-Sent Events for every query change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata files.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Core Semantic Router</title>`.

The App UI tab is a three-pane layout: **left** is the query list (status pill + domain chip + score chip), **centre** is the selected query's redacted payload + routing decision + score, **right** is the chosen specialist's answer + guardrail verdict + published response (or rejections when `BLOCKED`).

Tab switching is attribute-based (`data-tab` / `data-panel`); no zombie panels in the DOM. The Architecture tab's mermaid diagrams carry the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels are not clipped.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied in `PiiSanitizer` Consumer): redacts employee IDs, SSNs, salary values, account numbers, and benefit plan references from the query body before any LLM sees it. The PII categories found are kept for audit; only the redacted text reaches the agents.
- **G1 — before-agent-invocation guardrail** on the routing decision: checks that the target domain is `HR` or `FINANCE` (not `AMBIGUOUS`), that confidence is `"high"` or `"medium"`, and that the caller session is authorized for that domain. Blocking — a failed check puts the query in `BLOCKED` before any specialist runs.
- **E1 — on-decision eval** (`eval-event`, on the routing decision): `RoutingEvalScorer` (Consumer) listens for `IntentRouted` events and calls `RoutingJudge` to produce a 1–5 score with a one-sentence rationale. Non-blocking — the score is metadata, not a gate; persistent low scores would be surfaced as a system-level alert in a deployed setting.

## 9. Agent prompts

- `IntentRouterAgent` → `prompts/intent-router-agent.md`. Typed classifier; returns one of `HR`, `FINANCE`, `AMBIGUOUS`; defaults to `AMBIGUOUS` under ambiguity.
- `HrSpecialist` → `prompts/hr-specialist.md`. Owns the `ANSWER` task for HR queries. Never quotes raw employee data; escalates outside its authority.
- `FinanceSpecialist` → `prompts/finance-specialist.md`. Owns the `ANSWER` task for Finance queries. Never echoes account numbers; escalates outside its authority.
- `RoutingJudge` → `prompts/routing-judge.md`. Grades a routing decision against a 1–5 rubric with one-sentence rationale.
- `RoutingGuardrail` → `prompts/routing-guardrail.md`. Returns a `GuardrailVerdict { authorized, rejections }`. Conservative — borderline decisions are blocked.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips an HR-flavoured query → classified `HR` → guardrail passes → answered by `HrSpecialist` → published.
2. **J2** — Simulator drips a Finance query → classified `FINANCE` → guardrail passes → answered by `FinanceSpecialist` → published.
3. **J3** — An ambiguous query classifies as `AMBIGUOUS` and lands in `ESCALATED` without any specialist invocation.
4. **J4** — A query whose confidence falls below threshold fails the before-agent-invocation guardrail; the query lands in `BLOCKED` with the rejection listed; no specialist runs.
5. **J5** — Every classified query carries a `RoutingScore` (1–5) and rationale within ~10 s of the routing decision.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named core-semantic-router demonstrating the
handoff-routing × hr-recruiting cell.
Runs out of the box (in-process simulated inbound stream; no real HR or
Finance system integration).
Maven group io.akka.samples. Maven artifact
handoff-routing-hr-recruiting-intent-router-specialists. Java package
io.akka.samples.coresemanticrouter. Akka 3.6.0. HTTP port 9189.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) IntentRouterAgent — classifier. System
  prompt loaded from prompts/intent-router-agent.md. Input:
  SanitizedQuery{redactedSubject, redactedBody,
  piiCategoriesFound: List<String>}. Output: RoutingDecision{domain:
  QueryDomain (HR/FINANCE/AMBIGUOUS), confidence:
  "high"|"medium"|"low", reason: String}. Defaults to AMBIGUOUS under
  uncertainty.

- 1 AutonomousAgent HrSpecialist — definition() with
  capability(TaskAcceptance.of(ANSWER).maxIterationsPerTask(3)). System
  prompt from prompts/hr-specialist.md. Input: SanitizedQuery +
  RoutingDecision. Output: QueryAnswer{answerBody, action: AnswerAction,
  specialistTag = "hr", answeredAt}. Never quotes raw employee data;
  sets action=ESCALATED when outside authority.

- 1 AutonomousAgent FinanceSpecialist — definition() with
  capability(TaskAcceptance.of(ANSWER).maxIterationsPerTask(3)). System
  prompt from prompts/finance-specialist.md. Same input shape;
  specialistTag = "finance".

- 1 Agent (typed) RoutingJudge — judge. System prompt from
  prompts/routing-judge.md. Input: SanitizedQuery + RoutingDecision.
  Output: RoutingScore{score: int 1–5, rationale: String, scoredAt:
  Instant}.

- 1 Agent (typed) RoutingGuardrail — typed rubric check. System prompt
  from prompts/routing-guardrail.md. Input: RoutingDecision + caller
  context (requesterId, channel). Output: GuardrailVerdict{authorized:
  boolean, rejections: List<String>, rubricVersion: String}. Used by
  RouterWorkflow before specialist invocation; blocking.

- 1 Workflow RouterWorkflow per queryId. Steps:
    classifyStep -> guardrailStep -> {hrStep | financeStep | escalateStep}
                 -> publishStep
  classifyStep calls componentClient.forAgent().inSession(queryId)
    .method(IntentRouterAgent::classify).invoke(sanitized). On success
    emits IntentClassified via QueryEntity.recordClassification.
  guardrailStep calls componentClient.forAgent().inSession(queryId)
    .method(RoutingGuardrail::check).invoke(decision, callerCtx).
    On verdict.authorized=true proceed to route step (emits IntentRouted).
    On verdict.authorized=false emit QueryBlocked (terminal BLOCKED) and end.
  hrStep / financeStep call forAutonomousAgent(<Specialist>.class, queryId)
    .runSingleTask(TaskDef.instructions(buildPrompt(sanitized, routing)))
    returning a taskId, then forTask(taskId).result(RouterTasks.ANSWER)
    to block on the typed QueryAnswer. On success emits QueryAnswered.
  publishStep emits AnswerPublished (terminal ANSWERED).
  escalateStep emits QueryEscalated (terminal ESCALATED) when domain is
    AMBIGUOUS.
  Override settings() with stepTimeout(Duration.ofSeconds(20)) on
    classifyStep and guardrailStep, stepTimeout(Duration.ofSeconds(60))
    on hrStep, financeStep, and publishStep.
    defaultStepRecovery(maxRetries(2).failoverTo(RouterWorkflow::error)).

- 2 EventSourcedEntities:
    * QueryQueue — append-only audit log. Command receive(IncomingQuery)
      emits InboundQueryReceived{incoming}. No mutable state beyond a
      counter; commands are idempotent on incoming.queryId.
    * QueryEntity (one per queryId) — full per-query lifecycle. State
      Query{queryId, incoming: IncomingQuery, Optional<SanitizedQuery>
      sanitized, Optional<RoutingDecision> routing,
      Optional<GuardrailVerdict> guardrail, Optional<QueryAnswer> answer,
      Optional<RoutingScore> routingScore,
      Optional<String> escalationReason, QueryStatus status,
      Instant createdAt, Optional<Instant> finishedAt}.
      QueryStatus enum: RECEIVED, SANITIZED, CLASSIFIED, AUTHORIZED,
      ROUTED_HR, ROUTED_FINANCE, ANSWERED, BLOCKED, ESCALATED.
      Events: QueryRegistered, QuerySanitized, IntentClassified,
      IntentRouted, GuardrailVerdictAttached, QueryAnswered, QueryBlocked,
      QueryEscalated, RoutingScored.
      Commands: registerIncoming, attachSanitized, recordClassification,
      recordRouting, recordGuardrailVerdict, recordAnswer, publish, block,
      escalate, recordRoutingScore, getQuery.
      emptyState() returns Query.initial("") with no commandContext()
      reference.

- 2 Consumers:
    * PiiSanitizer subscribed to QueryQueue events; for each
      InboundQueryReceived applies a regex+heuristic redaction pipeline
      (employee IDs matching EMP-\w+, SSN patterns, salary values,
      account numbers matching ACCT-\w+, benefit-plan codes matching
      PLN-\d+) to subject + body, builds SanitizedQuery with
      piiCategoriesFound, and calls QueryEntity.registerIncoming then
      attachSanitized for the queryId; then starts a RouterWorkflow with
      queryId as the workflow id.
    * RoutingEvalScorer subscribed to QueryEntity events; on IntentRouted
      invokes RoutingJudge.score(sanitized, decision) and calls
      QueryEntity.recordRoutingScore(queryId, score). On any other event
      type, no-op. Use componentClient — do NOT call the agent from a
      TimedAction.

- 1 View QueryView with row type QueryRow (mirrors Query; uses Optional<T>
  for every nullable lifecycle field per Lesson 6). Table updater consumes
  QueryEntity events. ONE query getAllQueries SELECT * AS queries FROM
  query_view. No WHERE domain or WHERE status filter (Akka cannot
  auto-index enum columns) — filter client-side in callers.

- 1 TimedAction QuerySimulator — every 30s, reads next line from
  src/main/resources/sample-events/queries.jsonl (loops at EOF) and
  calls QueryQueue.receive with a fresh queryId (UUID).

- 2 HttpEndpoints:
    * RouterEndpoint at /api with GET /queries (list from
      QueryView.getAllQueries, filter client-side by ?domain and ?status
      query params), GET /queries/{id},
      POST /queries (body IncomingQuery minus queryId/receivedAt — server
      assigns),
      GET /queries/sse (serverSentEventsForView over getAllQueries), and
      three /api/metadata/{readme,risk-survey,eval-matrix} endpoints
      serving the YAML/MD files from src/main/resources/metadata/.
    * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
      static-resources/*.

Companion files:
- RouterTasks.java declaring the task constants: ANSWER
  (resultConformsTo QueryAnswer.class, description "Answer the HR or
  Finance query end-to-end and return a typed QueryAnswer").
- Domain records IncomingQuery, SanitizedQuery, RoutingDecision,
  GuardrailVerdict, QueryAnswer, RoutingScore, and the Query entity state.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9189 and the three model-provider
  blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/queries.jsonl with 9 canned lines
  (3 HR-flavoured, 3 Finance-flavoured, 2 AMBIGUOUS-flavoured, 1
  designed to trip the guardrail with confidence "low" on a known domain).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml,
  README.md (copies of the root-level files for the endpoint to serve
  from classpath).
- eval-matrix.yaml at the project root with 3 controls: S1 sanitizer pii,
  G1 guardrail before-agent-invocation, E1 eval-event on-decision-eval.
  Matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root with purpose.primary_function =
  employee-self-service, data.data_classes.pii = true,
  decisions.authority_level = autonomous-with-guardrail,
  oversight.human_in_loop = false,
  data.pii_handled_by_sanitizer_before_llm = true,
  failure.failure_modes including "wrong-domain-routing",
  "out-of-scope-answer", "pii-leakage-via-llm",
  "unauthorized-domain-access"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/intent-router-agent.md, prompts/hr-specialist.md,
  prompts/finance-specialist.md, prompts/routing-judge.md,
  prompts/routing-guardrail.md loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: Core Semantic
  Router", prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained
  file (no ui/, no npm). Five tabs matching the formal exemplar. App UI
  tab uses a three-column layout (left = query list with domain chip +
  status pill + score chip; centre = redacted query + routing block;
  right = specialist answer + guardrail verdict + published response or
  rejections). Browser title exactly:
  <title>Akka Sample: Core Semantic Router</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment
  for ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If
  exactly one is set, default application.conf's model-provider to match
  and proceed silently.
- If none is set, ask the user how to source the key, offering five
  options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the
        JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run
        time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only
  the REFERENCE (env-var name, file path, secrets URI); the value lives
  in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The message
  must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java implementing the ModelProvider interface
  with a per-agent dispatch on the agent class or Task<R> id. Each branch
  reads src/main/resources/mock-responses/<agent>.json, picks one entry
  pseudo-randomly per call (seeded by queryId so reruns are deterministic),
  and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    intent-router-agent.json — 12 RoutingDecision entries spanning HR
      (leave policy, benefits enrollment, onboarding, performance review,
      workplace policy), FINANCE (expense reimbursement, budget approval,
      payroll discrepancy, invoice processing), and AMBIGUOUS (vague
      messages, mixed content, off-domain). Confidence + a one-sentence
      reason on each.
    hr-specialist.json — 8 QueryAnswer entries: 5 with action
      POLICY_CITED or PROCESS_EXPLAINED (within authority), 1 with
      action ESCALATED (question requires HR director approval), 1 with
      action REFERRED (routing to legal for policy interpretation), 1
      designed to trip the guardrail (echoes a [REDACTED] token).
    finance-specialist.json — 8 QueryAnswer entries: 5 with action
      PROCESS_EXPLAINED or POLICY_CITED, 1 with action ESCALATED
      (amount exceeds reimbursement authority), 1 with action
      REFERRED (routing to audit), 1 designed to trip the guardrail
      (quotes a specific account number that was not in the sanitized
      input).
    routing-judge.json — 10 RoutingScore entries, score 1–5, one-sentence
      rationale matching the rubric (domain-correctness /
      confidence-calibration / reason-quality).
    routing-guardrail.json — 10 GuardrailVerdict entries. 7 with
      authorized=true and empty rejections. 3 with authorized=false and
      one rejection each ("low-confidence", "ambiguous-domain",
      "unauthorized-channel"). The mock should match the guardrail-
      tripping entry above when called for the same queryId.
- A MockModelProvider.seedFor(queryId) helper makes per-query selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- (Lesson 1) AutonomousAgent is never silently downgraded to Agent.
  HrSpecialist and FinanceSpecialist both extend
  akka.javasdk.agent.autonomous.AutonomousAgent and declare definition().
- (Lesson 4) Workflow step timeouts overridden via settings(): classifyStep
  20s, guardrailStep 20s, hrStep / financeStep / publishStep 60s each.
- (Lesson 6) Every nullable lifecycle field on Query is Optional<T>. The
  QueryView row type uses the same Optional wrapping.
- (Lesson 7) RouterTasks.java declares the ANSWER Task<QueryAnswer>
  constant. Both specialists' definition().capability(
  TaskAcceptance.of(ANSWER)...) reference it.
- (Lesson 8) Model names verified against current lineup: claude-sonnet-4-6,
  gpt-4o, gemini-2.5-flash.
- (Lesson 9) Run command is "/akka:build" everywhere. No "mvn akka:run".
- (Lesson 10) Port 9189 in application.conf; not 9000.
- (Lesson 11) No source.platform string anywhere user-facing.
- (Lesson 12) Static UI fits in 1080px content column with no horizontal
  scroll.
- (Lesson 13) Integration tier label is "Runs out of the box" — never T1.
- (Lesson 23) No competitor brand names in any user-facing text.
- (Lesson 24) static-resources/index.html includes the mermaid CSS
  overrides AND theme variables (state-diagram label colour white,
  edge-label foreignObject overflow:visible,
  transitionLabelColor #cccccc).
- (Lesson 25) API key sourcing follows the five-option protocol above.
- (Lesson 26) Tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. No "hidden" zombie panels.
- The PiiSanitizer runs INSIDE a Consumer before any LLM call — not
  inside an Agent's prompt and not after the LLM has seen the raw
  payload.
- The RoutingEvalScorer Consumer reacts to IntentRouted events and calls
  RoutingJudge via componentClient.forAgent(). It does NOT modify the
  workflow flow — the eval is out-of-band metadata.
- The guardrail step happens BEFORE any specialist is invoked. A blocked
  routing decision never reaches any specialist — only as a
  "blocked + rejections" surface for the operator.
- No forbidden words in user-facing text: shape, minimal, smaller,
  complex, Akka SDK in narrative, T1/T2/T3/T4, deferred, use,
  use, marketing tone, competitor brand names.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local key-source reference written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
