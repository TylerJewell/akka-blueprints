# SPEC — structured-output-reflection

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Reliable Structured Generation (Reflection Loop).
**One-line pitch:** Submit a schema and a prompt; a generator agent produces structured JSON output; a critic agent validates it and scores it for quality; the two iterate until the output passes or the loop hits its retry ceiling.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`GeneratorAgent`) and a reviewer agent (`CriticAgent`), feeding each validation report back into the next generation attempt until the output passes both schema validation and semantic quality checks, or a halt ends the loop. The blueprint also demonstrates three governance mechanisms — an **eval-event** that records every cycle's verdict for downstream quality measurement, an **output guardrail** that gates each generated document against deterministic schema validation before the semantic critic runs, and a **halt** that ends the loop gracefully at the retry ceiling without leaving the generation in a degenerate state.

## 3. User-facing flows

The user opens the App UI tab and submits a generation request (a schema name and a natural-language prompt describing the desired content).

1. The system creates a `Generation` record in `GENERATING` and starts a `ReflectionWorkflow`.
2. The Generator produces attempt #1: a JSON document targeting the named schema.
3. The schema guardrail validates the JSON structurally. Documents that fail schema parsing are short-circuited back to the Generator with a deterministic feedback note; they never reach the Critic.
4. The Critic scores the semantically-valid document against a fixed rubric (schema conformance, field completeness, value plausibility, internal consistency) and returns either `PASS` with a one-line rationale, or `REVISE` with a typed `ValidationNotes` payload (three bullets at most).
5. On `PASS`, the workflow transitions the generation to `PASSED` with the winning attempt's document and the critic's rationale.
6. On `REVISE`, the workflow records the attempt, the guardrail verdict, the validation notes, and the critic's verdict on the entity, then calls the Generator again with the notes attached. The Generator produces attempt #2.
7. If the loop reaches `maxAttempts` (default 4) without a `PASS`, the halt mechanism activates: the workflow ends with `FAILED_FINAL`, the highest-scoring attempt is preserved on the entity along with every critique for audit, and an `EvalRecorded` event captures the rejection.

A `RequestSimulator` (TimedAction) drips a canned generation request every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `GeneratorAgent` | `AutonomousAgent` | Produces a structured JSON document for the named schema; accepts prior validation notes on revision calls. | `ReflectionWorkflow` | returns `GeneratedOutput` to workflow |
| `CriticAgent` | `AutonomousAgent` | Scores a valid document against the rubric; returns `PASS` or `REVISE` with notes. | `ReflectionWorkflow` | returns `ValidationReport` to workflow |
| `ReflectionWorkflow` | `Workflow` | Runs the generate → guardrail → validate → revise loop; halts at the ceiling. | `GenerationEndpoint`, `SubmissionConsumer` | `GenerationEntity` |
| `GenerationEntity` | `EventSourcedEntity` | Holds the generation lifecycle, every attempt, every critique, and the final outcome. | `ReflectionWorkflow` | `GenerationsView` |
| `SubmissionQueue` | `EventSourcedEntity` | Logs each submitted request for replay and audit. | `GenerationEndpoint`, `RequestSimulator` | `SubmissionConsumer` |
| `GenerationsView` | `View` | List-of-generations read model. | `GenerationEntity` events | `GenerationEndpoint` |
| `SubmissionConsumer` | `Consumer` | Subscribes to `SubmissionQueue` events; starts a workflow per submission. | `SubmissionQueue` events | `ReflectionWorkflow` |
| `RequestSimulator` | `TimedAction` | Drips a sample request every 60 s from `sample-events/generation-requests.jsonl`. | scheduler | `SubmissionQueue` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `GenerationsView`, records an `EvalRecorded` event for any cycle that completed since the last tick. | scheduler | `GenerationEntity` |
| `GenerationEndpoint` | `HttpEndpoint` | `/api/generations/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `GenerationsView`, `SubmissionQueue`, `GenerationEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record GenerationRequest(String schemaName, String prompt, String requestedBy) {}

record GeneratedOutput(String json, int tokenCount, boolean parseable, Instant generatedAt) {}

record GuardrailVerdict(boolean passed, String reasonCode, String detail) {}

record ValidationNotes(List<String> bullets, String overallRationale) {}

record ValidationReport(
    CriticVerdict verdict,
    ValidationNotes notes,
    int score,
    Instant evaluatedAt
) {}

record Attempt(
    int attemptNumber,
    GeneratedOutput output,
    GuardrailVerdict guardrail,
    Optional<ValidationReport> report
) {}

record Generation(
    String generationId,
    String schemaName,
    String prompt,
    int maxAttempts,
    GenerationStatus status,
    List<Attempt> attempts,
    Optional<Integer> passedAttemptNumber,
    Optional<String> passedJson,
    Optional<String> failureReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum GenerationStatus { GENERATING, VALIDATING, PASSED, FAILED_FINAL }

enum CriticVerdict { PASS, REVISE }
```

### Events (on `GenerationEntity`)

`GenerationCreated`, `AttemptGenerated`, `AttemptGuardrailVerdictRecorded`, `AttemptValidated`, `GenerationPassed`, `GenerationFailedFinal`, `EvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/generations` — body `{ schemaName, prompt, requestedBy? }` → `{ generationId }`. Starts a workflow.
- `GET /api/generations` — list all generations. Optional `?status=GENERATING|VALIDATING|PASSED|FAILED_FINAL`.
- `GET /api/generations/{id}` — one generation (including every attempt and every validation report).
- `GET /api/generations/sse` — server-sent events stream of every generation change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Reliable Structured Generation"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, guardrail = red, halt = red).
- **App UI** — form to submit a generation request, live list of generations with status pills, click-to-expand per-attempt timeline showing each output, the guardrail verdict, the critic's verdict, and the validation notes.

Browser title: `<title>Akka Sample: Reliable Structured Generation</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — schema guardrail** (`after-llm-response` on `GeneratorAgent`): a deterministic structural check that the output parses as valid JSON and satisfies the named schema's required fields. Documents that fail this check are short-circuited back to the Generator with a structured feedback note (`reasonCode = SCHEMA_INVALID`); they never reach the semantic Critic. Enforcement: blocking.
- **E1 — eval-event** (`on-decision-eval`): every cycle's validation report is recorded as an `EvalRecorded` event with `{ attemptNumber, verdict, score, schemaFailed }`. The `EvalSampler` TimedAction is the canonical writer; the workflow itself also emits an event on terminal transitions. Enforcement: non-blocking. The events surface in the App UI's per-attempt timeline and in `/api/generations/{id}`.
- **HT1 — halt** (`graceful-degradation`): when the loop reaches `maxAttempts` without a `PASS`, the workflow ends with `GenerationFailedFinal`. The entity preserves every attempt, every validation report, the highest-scoring attempt's document, and a structured failure reason. The system never deletes outputs or terminates abruptly; the halt is observable end-to-end. Enforcement: system-level.

## 9. Agent prompts

- `GeneratorAgent` → `prompts/generator.md`. Produces a structured JSON document for the named schema; on a revision call, takes the prior `ValidationNotes` as input and produces a corrected document.
- `CriticAgent` → `prompts/critic.md`. Scores a schema-valid document against the fixed rubric; returns `PASS` with a one-line rationale or `REVISE` with three short bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a generation request; generation progresses `GENERATING` → `VALIDATING` → `PASSED` within the retry ceiling; the App UI shows every attempt's output and validation report.
2. **J2 — halt at ceiling** — Submit a request whose schema validation is impossible (test mode forces the Critic to `REVISE` every attempt); generation progresses through every attempt and lands in `FAILED_FINAL` with the best output preserved and a structured failure reason.
3. **J3 — schema guardrail block** — The Generator's first attempt produces structurally invalid JSON; the guardrail short-circuits with `reasonCode = SCHEMA_INVALID`, the Generator re-attempts with structured feedback, the cycle continues.
4. **J4 — eval-event timeline** — The expanded view of any generation shows one `EvalRecorded` event per attempt and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named structured-output-reflection demonstrating the
evaluator-optimizer × general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id
evaluator-optimizer-general-structured-output-reflection.
Java package io.akka.samples.reliablestructuredgenerationreflectionloop.
Akka 3.6.0. HTTP port 9988.

Components to wire (exactly):
- 2 AutonomousAgents:
  * GeneratorAgent — definition() with
    capability(TaskAcceptance.of(GENERATE).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REVISE_OUTPUT).maxIterationsPerTask(3)).
    System prompt loaded from prompts/generator.md. Returns
    GeneratedOutput{json, tokenCount, parseable, generatedAt} for both GENERATE
    and REVISE_OUTPUT. The REVISE_OUTPUT task takes (schemaName, prompt,
    priorOutput, ValidationNotes) as inputs.
  * CriticAgent — definition() with
    capability(TaskAcceptance.of(VALIDATE).maxIterationsPerTask(2)).
    System prompt from prompts/critic.md. Returns
    ValidationReport{verdict, notes, score, evaluatedAt} where verdict is the
    CriticVerdict enum (PASS | REVISE) and score is a 1–5 integer rubric.

- 1 Workflow ReflectionWorkflow with steps:
    startStep -> generateStep -> guardrailStep ->
    [guardrail FAIL? generateStep again with structured feedback :
    validateStep] ->
    [verdict PASS? passStep : (attemptCount < maxAttempts ?
       generateStep with notes attached : failStep)] -> END.
  generateStep calls forAutonomousAgent(GeneratorAgent.class, generationId)
    .runSingleTask(GENERATE or REVISE_OUTPUT) then
    forTask(taskId).result(GENERATE or REVISE_OUTPUT).
  validateStep calls forAutonomousAgent(CriticAgent.class, generationId)
    .runSingleTask(VALIDATE).
  passStep emits GenerationPassed.
  failStep emits GenerationFailedFinal with the highest-scoring attempt's
    document as best-of and a structured failureReason.
  Override settings() with stepTimeout(60s) on generateStep and validateStep,
  and defaultStepRecovery(maxRetries(2).failoverTo(failStep)).
  guardrailStep is a pure-function step (no LLM call): checks that
    output.parseable() == true and required fields for the named schema are
    present. On FAIL, emits AttemptGuardrailVerdictRecorded with
    verdict.passed = false and reasonCode = "SCHEMA_INVALID", then transitions
    back to generateStep with a structured feedback ValidationNotes
    ("Output is not valid JSON or is missing required schema fields; correct and
    resubmit.").

- 1 EventSourcedEntity GenerationEntity holding state Generation{generationId,
  schemaName, prompt, maxAttempts, GenerationStatus status, List<Attempt>
  attempts, Optional<Integer> passedAttemptNumber, Optional<String> passedJson,
  Optional<String> failureReason, Instant createdAt, Optional<Instant>
  finishedAt}. GenerationStatus enum: GENERATING, VALIDATING, PASSED,
  FAILED_FINAL. Events: GenerationCreated, AttemptGenerated,
  AttemptGuardrailVerdictRecorded, AttemptValidated, GenerationPassed,
  GenerationFailedFinal, EvalRecorded. Commands: createGeneration, recordOutput,
  recordGuardrail, recordValidation, pass, failFinal, recordEval, getGeneration.
  emptyState() returns Generation.initial("", "", 4) with no commandContext()
  reference. Event-applier wraps lifecycle fields with Optional.of(...).

- 1 EventSourcedEntity SubmissionQueue with command
  enqueueRequest(schemaName, prompt, requestedBy) emitting
  RequestSubmitted{generationId, schemaName, prompt, requestedBy, submittedAt}.

- 1 View GenerationsView with row type GenerationRow (mirrors Generation; the
  attempts list is preserved as-is — the list is bounded at maxAttempts so size
  stays reasonable). Table updater consumes GenerationEntity events. ONE query
  getAllGenerations SELECT * AS generations FROM generations_view. No WHERE
  status filter — caller filters client-side because Akka cannot auto-index enum
  columns (Lesson 2).

- 1 Consumer SubmissionConsumer subscribed to SubmissionQueue events; on
  RequestSubmitted starts a ReflectionWorkflow with the generationId as the
  workflow id.

- 2 TimedActions:
  * RequestSimulator — every 60s, reads next line from
    src/main/resources/sample-events/generation-requests.jsonl and calls
    SubmissionQueue.enqueueRequest.
  * EvalSampler — every 30s, queries GenerationsView.getAllGenerations, finds
    generations with a validated attempt that has not yet been recorded as an
    EvalRecorded event, and calls GenerationEntity.recordEval(attemptNumber,
    verdict, score, schemaFailed). Idempotent per (generationId, attemptNumber).

- 2 HttpEndpoints:
  * GenerationEndpoint at /api with POST /generations, GET /generations,
    GET /generations/{id}, GET /generations/sse, and three /api/metadata/*
    endpoints serving the YAML/MD files from
    src/main/resources/metadata/. The POST /generations body is
    {schemaName, prompt, requestedBy?}; missing requestedBy defaults to
    "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- GenerationTasks.java declaring three Task<R> constants: GENERATE (resultConformsTo
  GeneratedOutput), REVISE_OUTPUT (GeneratedOutput), VALIDATE (ValidationReport).
- Domain records GeneratedOutput, GuardrailVerdict, ValidationNotes,
  ValidationReport, Attempt, Generation; enums GenerationStatus, CriticVerdict.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9988 and akka.javasdk.agent model-provider
  blocks for anthropic (claude-sonnet-4-6), openai (gpt-4o),
  googleai-gemini (gemini-2.5-flash), each api-key read from the canonical env
  vars (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  structured-output-reflection.max-attempts = 4, overridable by env var.
- src/main/resources/sample-events/generation-requests.jsonl with 8 canned
  request lines, each shaped {"schemaName":"product","prompt":"..."}.
  Schemas: product (name, price, sku, description, category),
  event-record (title, startDate, endDate, location, capacity),
  contact (firstName, lastName, email, phone, company).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (G1 schema guardrail
  after-llm-response, E1 eval-event on-decision-eval, HT1 halt
  graceful-degradation) and a matching simplified_view list. No
  regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = structured-data-generation,
  decisions.authority_level = draft-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/generator.md, prompts/critic.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: Reliable Structured
  Generation", one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration
  section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN imports
  for markdown and YAML libs are acceptable. Five tabs matching the formal
  exemplar: Overview (eyebrow + headline + no subtitle + Try it / How it works /
  Components / API contract cards); Architecture (4 mermaid diagrams +
  click-to-expand component table); Risk Survey (7 sub-tabs from governance.html
  with answers populated from risk-survey.yaml; unanswered .qb opacity 0.45);
  Eval Matrix (5-column ID/Control/Mechanism/Implementation/Source table with
  click-to-expand rows); App UI (form + live list with status pills,
  click-to-expand per-attempt timeline). Browser title exactly:
  <title>Akka Sample: Reliable Structured Generation</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value passed via MCP tool environment param.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java implementing ModelProvider with per-agent
  dispatch on agent class name or Task<R> id. Each branch reads a JSON file
  from src/main/resources/mock-responses/<agent-name>.json.
- Per-agent mock-response shapes for THIS blueprint:
    generator.json — 6 GeneratedOutput entries. Three are well-formed product/
      event-record/contact JSON documents. Two are revision attempts that
      incorporate prior notes (corrected field values, added missing fields).
      One is intentionally invalid JSON (missing closing brace) to exercise
      the schema guardrail in J3.
    critic.json — 6 ValidationReport entries. Three return verdict=PASS with
      score=4 or 5. Three return verdict=REVISE with score=2 or 3 and three
      ValidationNotes bullets ("price field is a string, expected number",
      "required field 'sku' is absent", "description exceeds recommended length").
- A MockModelProvider.seedFor(generationId, attemptNumber) helper makes the
  selection deterministic per (generationId, attemptNumber).

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. GeneratorAgent
  and CriticAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on the Generation row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: GenerationTasks.java is mandatory; generating GeneratorAgent or
  CriticAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build, never mvn akka:run.
- Lesson 10: HTTP port is 9988, declared in application.conf
  dev-mode.http-port.
- Lesson 11: source.platform is corpus-internal; the generated UI never
  surfaces a competitor brand name.
- Lesson 12: the App UI fits the 1080px content column with no horizontal
  scroll.
- Lesson 13: integration tier is shown as "Runs out of the box" — never
  T1/T2/T3/T4, never the word "deferred".
- Lesson 23: forbidden words (shape, minimal, smaller, complex, Akka SDK in
  narrative, marketing tone, competitor brand names) do not appear in any
  user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND
  theme variables for state-diagram label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc.
- Lesson 25: NEVER write the key value to disk. application.conf records only
  ${?VAR_NAME} substitution; Bootstrap.java fails fast if the reference does
  not resolve.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER
  by NodeList index. The DOM contains exactly five <section class="tab-panel">
  elements; removed panels are deleted from the HTML, not hidden with
  display:none.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars and the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
