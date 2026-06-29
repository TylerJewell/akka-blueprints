# SPEC — hr-assistant

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** HR Assistant.
**One-line pitch:** An employee submits a natural-language HR question; one AI agent looks up the relevant policy sections and job postings (passed as a query context attachment, never as inline prompt text) and returns a structured `PolicyAnswer` — a direct response, a list of policy citations, and an applicability verdict.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the HR-recruiting domain. One `HrPolicyAgent` (AutonomousAgent) carries the entire answer decision; the surrounding components prepare its input and audit its output. Three governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw query submission and the agent call — so the model never sees employee identifiers, email addresses, or personnel file numbers.
- A **special-category sanitizer** runs in the same Consumer pass, immediately after the PII pass — redacting health conditions, disability status, religious accommodations, union membership, and pregnancy status from both the query text and any resolved employee profile context. These fields are special-category under GDPR Article 9 / similar frameworks.
- A **before-agent-response guardrail** validates the agent's answer on every turn: well-formed JSON, every `citations[].policyId` references a real policy in the seeded corpus, every `applicabilityVerdict` is in the allowed set, and the answer text is non-empty. A malformed answer triggers a retry inside the same task.

The blueprint shows that even a conversational-style single-agent system benefits from layered governance: sanitization separates sensitive HR data from the model, and the guardrail ensures every employee-facing answer meets structural contract before it is persisted.

## 3. User-facing flows

The user opens the App UI tab.

1. The employee types a question into the **Query** textarea (or picks one of three seeded examples — a PTO balance inquiry, a parental leave question, a job-posting lookup).
2. The employee fills in an optional **Employee ID** (used to look up their profile from the seeded employee table). If omitted, the agent operates on the query alone without profile context.
3. The employee clicks **Submit query**. The UI POSTs to `/api/queries` and receives a `queryId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `SANITIZED` — the redacted query is visible in the card detail, with chips listing which PII and special-category field types were found and stripped.
5. Within ~10–30 s, the workflow's `answerStep` completes. The card transitions to `ANSWERING` then `ANSWER_RECORDED`. The answer appears: an `applicabilityVerdict` badge (APPLICABLE / NOT_APPLICABLE / PARTIALLY_APPLICABLE), an `answerText` paragraph, and a citations table (policy id, section reference, quoted passage).
6. The employee can submit another query; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `QueryEntity`, `QueryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `QueryEntity` | `EventSourcedEntity` | Per-query lifecycle: submitted → sanitized → answering → answer recorded → failed. Source of truth. | `QueryEndpoint`, `QuerySanitizer`, `QueryWorkflow` | `QueryView` |
| `QuerySanitizer` | `Consumer` | Subscribes to `QuerySubmitted` events; runs PII pass then special-category pass; calls `QueryEntity.attachSanitized`. | `QueryEntity` events | `QueryEntity` |
| `QueryWorkflow` | `Workflow` | One workflow per query. Steps: `awaitSanitizedStep` → `answerStep`. | started by `QuerySanitizer` once sanitized event lands | `HrPolicyAgent`, `QueryEntity` |
| `HrPolicyAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the query and relevant policy sections in the task definition and the sanitized employee profile + policy corpus as a task attachment; returns `PolicyAnswer`. | invoked by `QueryWorkflow` | returns answer |
| `QueryView` | `View` | Read model: one row per query for the UI. | `QueryEntity` events | `QueryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record PolicyCitation(
    String policyId,
    String sectionReference,
    String quotedPassage
) {}

enum ApplicabilityVerdict { APPLICABLE, NOT_APPLICABLE, PARTIALLY_APPLICABLE }

record PolicyAnswer(
    ApplicabilityVerdict applicabilityVerdict,
    String answerText,
    List<PolicyCitation> citations,
    Instant answeredAt
) {}

record EmployeeProfile(
    String employeeId,
    String department,
    String jobTitle,
    Instant hireDate,
    String workLocation
    // sensitive fields (health, disability, religion) are stripped before this record
    // reaches the agent; only the above fields survive the sanitizer
) {}

record SanitizedQueryContext(
    String redactedQueryText,
    List<String> piiCategoriesFound,
    List<String> specialCategoriesFound,
    Optional<EmployeeProfile> employeeProfile   // null when no employeeId supplied
) {}

record QueryRequest(
    String queryId,
    String rawQueryText,
    Optional<String> employeeId,
    Instant submittedAt
) {}

record HrQuery(
    String queryId,
    Optional<QueryRequest> request,
    Optional<SanitizedQueryContext> sanitized,
    Optional<PolicyAnswer> answer,
    QueryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus {
    SUBMITTED, SANITIZED, ANSWERING, ANSWER_RECORDED, FAILED
}
```

Events on `QueryEntity`: `QuerySubmitted`, `QuerySanitized`, `AnsweringStarted`, `AnswerRecorded`, `QueryFailed`.

Every nullable lifecycle field on the `HrQuery` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/queries` — body `{ rawQueryText, employeeId? }` → `{ queryId }`.
- `GET /api/queries` — list all queries, newest-first.
- `GET /api/queries/{id}` — one query.
- `GET /api/queries/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: HR Assistant</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted queries (status pill + applicability badge + age) and a right pane with the selected query's detail — sanitized query preview, employee profile chip (if resolved), answer text, and citations table.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `QuerySanitizer` Consumer): redacts emails, phone numbers, government identifiers, payment-card-like tokens, person names other than the querying employee's display name, postal addresses, and account-like identifiers from the raw query text and any resolved employee profile.
- **S2 — Special-category sanitizer** (`special-category`, same Consumer pass, runs after S1): redacts health conditions, disability status, religious accommodation requests, union membership indicators, and pregnancy or parental-status references. These fields are special-category data requiring explicit consent for AI processing.
- **G1 — before-agent-response guardrail**: runs on every turn of `HrPolicyAgent`. Asserts the candidate response is well-formed `PolicyAnswer` JSON, every `citations[].policyId` matches an id in the seeded policy corpus, the `applicabilityVerdict` is one of `{APPLICABLE, NOT_APPLICABLE, PARTIALLY_APPLICABLE}`, and `answerText` is non-empty. On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.

## 9. Agent prompts

- `HrPolicyAgent` → `prompts/hr-policy-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached policy corpus and employee profile, answer the employee's question directly, and return one `PolicyCitation` per supporting policy section.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Employee submits the PTO inquiry seed; within 30 s the answer appears with at least one citation per relevant policy section.
2. **J2** — The agent's first response on a query is intentionally malformed (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration produces a well-formed answer; the UI never displays the malformed response.
3. **J3** — A query context containing `disability: chronic back pain` is submitted; the LLM call log shows only `[REDACTED-SPECIAL-CATEGORY]`; the entity's `request.rawQueryText` retains the raw text for audit.
4. **J4** — A query containing `john.smith@example.com` and `EID-983421` is submitted; the LLM call log shows only `[REDACTED-EMAIL]` and `[REDACTED-PID]`; the entity's `request.rawQueryText` retains the raw text for audit.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named hr-assistant demonstrating the single-agent × hr-recruiting cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-hr-recruiting-hr-assistant. Java package io.akka.samples.inlineagenthrassistant.
Akka 3.6.0. HTTP port 9740.

Components to wire (exactly):

- 1 AutonomousAgent HrPolicyAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/hr-policy-agent.md>) and
  .capability(TaskAcceptance.of(ANSWER_HR_QUERY).maxIterationsPerTask(3)). The task receives
  the employee's query text as its instruction text and the sanitized context (policy corpus
  excerpt + employee profile) as a task ATTACHMENT (NOT as inline prompt text —
  Akka's TaskDef.attachment(name, contentBytes) is the canonical call).
  Output: PolicyAnswer{applicabilityVerdict: ApplicabilityVerdict, answerText: String,
  citations: List<PolicyCitation>, answeredAt: Instant}. The agent is configured with a
  before-agent-response guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries the response
  within its 3-iteration budget.

- 1 Workflow QueryWorkflow per queryId with two steps:
  * awaitSanitizedStep — polls QueryEntity.getQuery every 1s; on query.sanitized().isPresent()
    advances to answerStep. WorkflowSettings.stepTimeout 15s (sanitizer is in-process and fast).
  * answerStep — emits AnsweringStarted, then calls componentClient.forAutonomousAgent(
    HrPolicyAgent.class, "agent-" + queryId).runSingleTask(
      TaskDef.instructions(query.sanitized.redactedQueryText)
        .attachment("context.txt", buildContext(query.sanitized).getBytes())
    ) — returns a taskId, then forTask(taskId).result(ANSWER_HR_QUERY) to fetch the answer.
    On success calls QueryEntity.recordAnswer(answer). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(2).failoverTo(QueryWorkflow::error).
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity QueryEntity (one per queryId). State HrQuery{queryId: String,
  request: Optional<QueryRequest>, sanitized: Optional<SanitizedQueryContext>,
  answer: Optional<PolicyAnswer>, status: QueryStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. QueryStatus enum: SUBMITTED, SANITIZED, ANSWERING,
  ANSWER_RECORDED, FAILED. Events: QuerySubmitted{request}, QuerySanitized{sanitized},
  AnsweringStarted{}, AnswerRecorded{answer}, QueryFailed{reason}. Commands: submit,
  attachSanitized, markAnswering, recordAnswer, fail, getQuery. emptyState() returns
  HrQuery.initial("") with no commandContext() reference (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer QuerySanitizer subscribed to QueryEntity events; on QuerySubmitted runs two
  sequential redaction passes:
  Pass 1 (PII): regex+heuristic pipeline over rawQueryText — emails, phone numbers,
    government/employee-ID-like tokens, person names (heuristic), postal addresses,
    payment-card-like tokens. Outputs piiCategoriesFound.
  Pass 2 (special-category): keyword+context pipeline — health conditions, disability status,
    religious-accommodation phrases, union-membership indicators, pregnancy/parental-status
    phrases. Outputs specialCategoriesFound.
  Optionally resolves the EmployeeProfile from the seeded employee table if employeeId is
  present; strips any special-category fields from the profile before attaching it.
  Builds SanitizedQueryContext, then calls QueryEntity.attachSanitized(sanitized).
  After attachSanitized lands, the same Consumer starts a QueryWorkflow with
  id = "query-" + queryId.

- 1 View QueryView with row type QueryRow (mirrors HrQuery minus request.rawQueryText — the
  audit log keeps the raw; the view holds the sanitized form for the UI). Table updater
  consumes QueryEntity events. ONE query getAllQueries: SELECT * AS queries FROM query_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /queries (body {rawQueryText, employeeId?}; mints
    queryId; calls QueryEntity.submit; returns {queryId}), GET /queries (list from
    getAllQueries, sorted newest-first), GET /queries/{id} (one row), GET /queries/sse
    (Server-Sent Events forwarded from the view's stream-updates), and three /api/metadata/*
    endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- HrQueryTasks.java declaring one Task<R> constant: ANSWER_HR_QUERY = Task.name("Answer
  HR query").description("Read the attached policy corpus and employee profile context and
  produce a PolicyAnswer").resultConformsTo(PolicyAnswer.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records PolicyCitation, ApplicabilityVerdict, PolicyAnswer, EmployeeProfile,
  SanitizedQueryContext, QueryRequest, HrQuery, QueryStatus.

- PolicyAnswerGuardrail.java implementing the before-agent-response hook. Reads the candidate
  PolicyAnswer from the LLM response, runs the four checks listed in eval-matrix.yaml G1,
  and either passes the response through or returns Guardrail.reject(<structured-error>) to
  force the agent loop to retry.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9740 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The HrPolicyAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/policies.jsonl with 3 seeded policy documents:
  a PTO and leave policy (8 sections), a parental-leave and family-care policy (6 sections),
  and a job-posting lookup guide (5 sections describing how to search open roles).

- src/main/resources/sample-events/employees.jsonl with 5 synthetic employee profiles.
  Each profile contains only non-special-category fields (department, title, hire date,
  location). Deliberately includes 2 records that reference a health condition in a notes
  field so S2 has work to do.

- src/main/resources/sample-events/seed-queries.jsonl with 3 paired example queries:
  a PTO balance question (references employeeId EM-001), a parental leave eligibility question
  (plain query, no employeeId), and a job-posting search question. Each contains at least one
  PII string and one special-category phrase so both sanitizer passes execute.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (S1, S2, G1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  data.data_classes.special_category = true, pii_handled_by_sanitizer_before_llm = true,
  special_category_handled_by_sanitizer_before_llm = true,
  decisions.authority_level = assist-only (the agent's answer is informational, not a formal
  HR decision), oversight.human_in_loop = true (an HR professional reviews escalated queries),
  failure.failure_modes including "hallucinated-policy-citation", "stale-policy-reference",
  "pii-leakage-via-llm", "special-category-leakage-via-llm",
  "inapplicable-policy-cited"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/hr-policy-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: HR Assistant", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of query cards; right = selected-query detail with sanitized query preview,
  employee profile chip, answer text, and citations table).
  Browser title exactly: <title>Akka Sample: HR Assistant</title>. No subtitle on the
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(queryId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    answer-hr-query.json — 8 PolicyAnswer entries covering the three ApplicabilityVerdict
      values. Each entry has an answerText paragraph and a citations array with one
      PolicyCitation per relevant policy section (policyId, sectionReference, quotedPassage).
      Severities and applicability vary realistically. Plus 2 deliberately MALFORMED entries
      (one with a citations[].policyId not in the seeded corpus; one with a blank answerText)
      — the guardrail blocks both, exercising the retry path. The mock should select a
      malformed entry on the FIRST iteration of every 3rd query (modulo seed) so J2 is
      reproducible.
- A MockModelProvider.seedFor(queryId) helper makes per-query selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. HrPolicyAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion HrQueryTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (answerStep
  60s, awaitSanitizedStep 15s, error 5s).
- Lesson 6: every nullable lifecycle field on the HrQuery row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: HrQueryTasks.java with ANSWER_HR_QUERY = Task.name(...).description(...)
  .resultConformsTo(PolicyAnswer.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9740 declared explicitly in application.conf's
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (HrPolicyAgent). No
  second agent is wired anywhere — the sanitizer logic is deterministic code in QuerySanitizer.
- The query context is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated answerStep uses TaskDef.attachment(...) and not string
  interpolation into the instruction text.
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
