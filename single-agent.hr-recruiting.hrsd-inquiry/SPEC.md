# SPEC — hrsd-inquiry

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** HRSD Inquiry Agent.
**One-line pitch:** An employee types an HR question or service request; one AI agent reads the organization's policy catalog (passed as a task attachment, never as inline prompt text) and returns a grounded `InquiryResponse` — a policy-cited answer plus an optional `HrServiceRequest` the workflow can submit on the employee's behalf.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the hr-recruiting domain. One `HrInquiryAgent` (AutonomousAgent) carries the entire decision; the surrounding components only prepare its input and audit its output. Two governance mechanisms are wired around the agent:

- A **special-category data sanitizer** runs inside a Consumer between the raw inquiry submission and the agent call — so the model never sees health conditions, union membership status, religious preferences, pregnancy-related disclosures, or other special-category attributes the employee may have included in their message.
- A **before-agent-response guardrail** validates the agent's answer on every turn: every response must cite at least one policy id from the catalog, every `HrServiceRequest` (if present) must reference a valid request type, and the response must be parseable into `InquiryResponse`. A failing response triggers a retry inside the same task.

The blueprint shows that the single-agent pattern is not ungoverned — two independent checks sit on either side of the one decision-making LLM call, and HR-domain special-category data never reaches the model.

## 3. User-facing flows

The user opens the App UI tab.

1. The employee picks a **topic** from a dropdown (Benefits, Leave & Absence, Onboarding, Payroll, General HR) and types their inquiry into the **Message** textarea — or loads one of five seeded examples.
2. The employee optionally ticks **"Submit an HR service request if applicable"** (e.g., request a leave form, update a beneficiary).
3. The employee clicks **Ask HR**. The UI POSTs to `/api/inquiries` and receives an `inquiryId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `SCREENED` — the screened message is visible in the card detail with a small list of special-category data categories the screener found.
5. Within ~10–30 s, the workflow's `answerStep` completes. The card transitions to `ANSWERING` then `ANSWERED`. The response appears: the policy-grounded answer paragraph, a list of cited policy ids (linked to the policy catalog entry), and — if the agent produced one — an `HrServiceRequest` summary.
6. If a service request was produced and the employee ticked the submission checkbox, the `submitRequestStep` fires. The card transitions to `REQUEST_SUBMITTED` and shows a confirmation reference number.
7. The employee can submit another inquiry; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `InquiryEndpoint` | `HttpEndpoint` | `/api/inquiries/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `InquiryEntity`, `InquiryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `InquiryEntity` | `EventSourcedEntity` | Per-inquiry lifecycle: submitted → screened → answering → answered → request-submitted. Source of truth. | `InquiryEndpoint`, `SpecialCategoryScreener`, `InquiryWorkflow` | `InquiryView` |
| `SpecialCategoryScreener` | `Consumer` | Subscribes to `InquirySubmitted` events; redacts special-category data; calls `InquiryEntity.attachScreened`. | `InquiryEntity` events | `InquiryEntity` |
| `InquiryWorkflow` | `Workflow` | One workflow per inquiry. Steps: `awaitScreenedStep` → `answerStep` → (optional) `submitRequestStep`. | started by `SpecialCategoryScreener` once screened event lands | `HrInquiryAgent`, `InquiryEntity` |
| `HrInquiryAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the employee's screened message and the relevant policy catalog entries as a task attachment; returns `InquiryResponse`. | invoked by `InquiryWorkflow` | returns response |
| `ResponseGuardrail` | — (registered on agent) | before-agent-response hook; validates policy citations and response structure. | `HrInquiryAgent` candidate responses | agent loop |
| `InquiryView` | `View` | Read model: one row per inquiry for the UI. | `InquiryEntity` events | `InquiryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record PolicyRef(String policyId, String title, String section) {}

record InquiryRequest(
    String inquiryId,
    String employeeId,
    HrTopic topic,
    String rawMessage,
    boolean submitRequestIfApplicable,
    Instant submittedAt
) {}
enum HrTopic { BENEFITS, LEAVE_AND_ABSENCE, ONBOARDING, PAYROLL, GENERAL_HR }

record ScreenedMessage(
    String redactedMessage,
    List<String> specialCategoriesFound
) {}

record HrServiceRequest(
    String requestType,
    String description,
    Map<String, String> fields
) {}

record InquiryResponse(
    String answer,
    List<PolicyRef> citedPolicies,
    @Nullable HrServiceRequest serviceRequest,
    Instant answeredAt
) {}

record Inquiry(
    String inquiryId,
    Optional<InquiryRequest> request,
    Optional<ScreenedMessage> screened,
    Optional<InquiryResponse> response,
    Optional<String> serviceRequestRef,
    InquiryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum InquiryStatus {
    SUBMITTED, SCREENED, ANSWERING, ANSWERED, REQUEST_SUBMITTED, FAILED
}
```

Events on `InquiryEntity`: `InquirySubmitted`, `InquiryScreened`, `AnswerStarted`, `InquiryAnswered`, `ServiceRequestSubmitted`, `InquiryFailed`.

Every nullable lifecycle field on the `Inquiry` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/inquiries` — body `{ employeeId, topic, rawMessage, submitRequestIfApplicable: boolean }` → `{ inquiryId }`.
- `GET /api/inquiries` — list all inquiries, newest-first.
- `GET /api/inquiries/{id}` — one inquiry.
- `GET /api/inquiries/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: HRSD Inquiry Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted inquiries (status pill + topic badge + age) and a right pane with the selected inquiry's detail — screened message preview, cited policies list, answer paragraph, service request summary (if any), and confirmation reference (if submitted).

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — special-category data sanitizer** (`special-category`, applied inside `SpecialCategoryScreener` Consumer): redacts health conditions, disability-related disclosures, union membership references, religious preferences, pregnancy and parental-status disclosures, and any government identifier the employee might include. Records which categories were found.
- **G1 — before-agent-response guardrail**: runs on every turn of `HrInquiryAgent`. Asserts the candidate response is well-formed `InquiryResponse` JSON, every `citedPolicies[].policyId` exists in the policy catalog, and — if a `serviceRequest` is present — the `requestType` is in the allowed set. On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.

## 9. Agent prompts

- `HrInquiryAgent` → `prompts/hr-inquiry-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached policy catalog entries, answer the employee's screened question, and cite every policy it draws on.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Employee submits a benefits inquiry; within 30 s the answer appears with at least one policy citation.
2. **J2** — The agent's first response on an inquiry lacks any `citedPolicies` entries (mock LLM path) — the guardrail rejects it; the second iteration produces a properly-cited answer; the UI never displays the uncited response.
3. **J3** — An inquiry message containing `"I have Type 2 diabetes"` is submitted; the LLM call log shows only `[REDACTED-HEALTH]`; the entity's `request.rawMessage` retains the raw text.
4. **J4** — An employee asks for a leave request form and ticks the submission checkbox; the agent produces an `HrServiceRequest`; the workflow advances to `REQUEST_SUBMITTED` with a confirmation reference number visible in the UI.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named hrsd-inquiry demonstrating the single-agent × hr-recruiting cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-hr-recruiting-hrsd-inquiry. Java package io.akka.samples.hrsdinquiryagent.
Akka 3.6.0. HTTP port 9222.

Components to wire (exactly):

- 1 AutonomousAgent HrInquiryAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/hr-inquiry-agent.md>) and
  .capability(TaskAcceptance.of(ANSWER_INQUIRY).maxIterationsPerTask(3)). The task receives
  the employee's screened message as its instruction text and the relevant policy catalog
  entries as a task ATTACHMENT (NOT as inline prompt text — Akka's
  TaskDef.attachment(name, contentBytes) is the canonical call). Output:
  InquiryResponse{answer: String, citedPolicies: List<PolicyRef>, serviceRequest:
  HrServiceRequest (nullable), answeredAt: Instant}. The agent is configured with a
  before-agent-response guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries the response
  within its 3-iteration budget.

- 1 Workflow InquiryWorkflow per inquiryId with three steps:
  * awaitScreenedStep — polls InquiryEntity.getInquiry every 1s; on
    inquiry.screened().isPresent() advances to answerStep. WorkflowSettings.stepTimeout 15s
    (screener is in-process and fast).
  * answerStep — emits AnswerStarted, then calls componentClient.forAutonomousAgent(
    HrInquiryAgent.class, "inquirer-" + inquiryId).runSingleTask(
      TaskDef.instructions(inquiry.screened.redactedMessage)
        .attachment("policies.txt", loadPolicyCatalog(inquiry.request.topic).getBytes())
    ) — returns a taskId, then forTask(taskId).result(ANSWER_INQUIRY) to fetch the response.
    On success calls InquiryEntity.recordAnswer(response). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(2).failoverTo(InquiryWorkflow::error).
  * submitRequestStep — only fires if inquiry.request.submitRequestIfApplicable == true AND
    inquiry.response.serviceRequest != null. Calls a deterministic
    HrRequestSubmitter.submit(serviceRequest) (NOT an LLM call — returns a pseudo-random
    reference number seeded by inquiryId). Emits ServiceRequestSubmitted{ref: String}.
    WorkflowSettings.stepTimeout 10s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity InquiryEntity (one per inquiryId). State Inquiry{inquiryId: String,
  request: Optional<InquiryRequest>, screened: Optional<ScreenedMessage>,
  response: Optional<InquiryResponse>, serviceRequestRef: Optional<String>,
  status: InquiryStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  InquiryStatus enum: SUBMITTED, SCREENED, ANSWERING, ANSWERED, REQUEST_SUBMITTED, FAILED.
  Events: InquirySubmitted{request}, InquiryScreened{screened}, AnswerStarted{},
  InquiryAnswered{response}, ServiceRequestSubmitted{ref}, InquiryFailed{reason}.
  Commands: submit, attachScreened, markAnswering, recordAnswer, recordServiceRequest,
  fail, getInquiry. emptyState() returns Inquiry.initial("") with no commandContext()
  reference (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state
  and Optional.of(...) inside the event-applier.

- 1 Consumer SpecialCategoryScreener subscribed to InquiryEntity events; on
  InquirySubmitted runs a regex+heuristic pipeline over rawMessage redacting: health
  condition disclosures (e.g. "I have [condition]"), union membership references,
  religious/faith references, pregnancy and parental-status disclosures, disability
  references, and government identifiers. Computes the list of categories found, builds
  ScreenedMessage, then calls InquiryEntity.attachScreened(screened). After attachScreened
  lands, the same Consumer starts an InquiryWorkflow with id = "inquiry-" + inquiryId.

- 1 View InquiryView with row type InquiryRow (mirrors Inquiry minus request.rawMessage —
  the audit log keeps the raw; the view holds the screened form for the UI). Table updater
  consumes InquiryEntity events. ONE query getAllInquiries: SELECT * AS inquiries FROM
  inquiry_view. No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2);
  caller filters client-side.

- 2 HttpEndpoints:
  * InquiryEndpoint at /api with POST /inquiries (body
    {employeeId, topic, rawMessage, submitRequestIfApplicable: boolean};
    mints inquiryId; calls InquiryEntity.submit; returns {inquiryId}), GET /inquiries
    (list from getAllInquiries, sorted newest-first), GET /inquiries/{id} (one row), GET
    /inquiries/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- InquiryTasks.java declaring one Task<R> constant: ANSWER_INQUIRY = Task.name("Answer HR
  inquiry").description("Read the attached policy catalog and answer the employee inquiry
  with citations").resultConformsTo(InquiryResponse.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records PolicyRef, InquiryRequest, HrTopic, ScreenedMessage, HrServiceRequest,
  InquiryResponse, Inquiry, InquiryStatus.

- ResponseGuardrail.java implementing the before-agent-response hook. Reads the candidate
  InquiryResponse from the LLM response, runs the checks listed in eval-matrix.yaml G1,
  and either passes the response through or returns Guardrail.reject(<structured-error>) to
  force the agent loop to retry.

- HrRequestSubmitter.java — pure deterministic logic (no LLM). Inputs: HrServiceRequest
  and inquiryId. Outputs: String (confirmation reference). Reference is
  "HR-" + inquiryId.substring(0,8).toUpperCase() + "-" + String.valueOf(Math.abs(
  inquiryId.hashCode()) % 90000 + 10000). Javadoc documents the format.

- PolicyCatalogLoader.java — reads src/main/resources/policy-catalog/<topic>.txt at
  startup and exposes loadPolicyCatalog(HrTopic topic): String. The text files are plain
  prose policy summaries; InquiryWorkflow.answerStep calls this to build the attachment.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9222 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The HrInquiryAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/policy-catalog/ with five .txt files — benefits.txt, leave-and-absence.txt,
  onboarding.txt, payroll.txt, general-hr.txt — containing plausible synthetic HR policy
  summaries with named policy ids (e.g. "BEN-101: Medical Insurance Enrollment").

- src/main/resources/sample-events/inquiries.jsonl with 5 seeded inquiry examples:
  a benefits enrollment question, a FMLA leave request inquiry, an onboarding task question,
  a payroll discrepancy inquiry, and a general HR policy lookup. Each contains 1–2
  special-category disclosures so S1 has work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (S1, G1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.special_category_data = true,
  special_category_handled_by_screener_before_llm = true,
  decisions.authority_level = recommend-only (the agent's answer is advisory; the employee
  or an HR rep acts on it), oversight.human_in_loop = true, failure.failure_modes including
  "hallucinated-policy-citation", "missed-applicable-policy",
  "special-category-leakage-via-llm", "incorrect-service-request-type"; deployer fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/hr-inquiry-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: HRSD Inquiry Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of inquiry cards; right = selected-inquiry detail with screened message preview,
  cited policies list, answer paragraph, service request summary, and confirmation reference
  when submitted). Browser title exactly:
  <title>Akka Sample: HRSD Inquiry Agent</title>. No subtitle on the Overview tab.

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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(inquiryId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    answer-hr-inquiry.json — 8 InquiryResponse entries covering a mix of topics.
      Each entry has a one-paragraph answer, a non-empty citedPolicies list (2–4 PolicyRef
      entries with realistic policy ids like "BEN-101", "LEA-203"), and — in 3 of the 8
      entries — an HrServiceRequest with a valid requestType. Plus 2 deliberately MALFORMED
      entries (one with an empty citedPolicies list; one with a serviceRequest whose
      requestType is not in the allowed set) — the guardrail blocks both, exercising the
      retry path. The mock should select a malformed entry on the FIRST iteration of every
      3rd inquiry (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(inquiryId) helper makes per-inquiry selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. HrInquiryAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion InquiryTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (answerStep
  60s, awaitScreenedStep 15s, submitRequestStep 10s, error 5s).
- Lesson 6: every nullable lifecycle field on the Inquiry row record is Optional<T>. The
  view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: InquiryTasks.java with ANSWER_INQUIRY = Task.name(...).description(...)
  .resultConformsTo(InquiryResponse.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9222 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
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
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (HrInquiryAgent). The
  service-request submitter (HrRequestSubmitter.java) is deterministic and does NOT make
  an LLM call — keeping the pattern's "one agent" promise honest.
- The policy catalog is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated answerStep uses TaskDef.attachment(...) and not
  string interpolation into the instruction text.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent returns.
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
