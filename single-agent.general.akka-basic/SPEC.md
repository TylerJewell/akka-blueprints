# SPEC — akka-basic

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** AkkaBasic.
**One-line pitch:** A user submits a free-form prompt; one AI agent processes it and returns a typed `PromptResult` — output text, a confidence score, and a category label — wrapped in a durable workflow that retries on transient failure and records every outcome in an event-sourced entity.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in its most general form. One `PromptAgent` (AutonomousAgent) carries the entire LLM call; the surrounding components handle submission, durability, and output validation. One governance mechanism is wired around the agent:

- An **after-agent-response guardrail** validates the agent's result on every turn: the `outputText` field is non-empty, the `confidenceScore` is in `[0.0, 1.0]`, and the `category` value is in the allowed set. A result that fails any check triggers a retry inside the same task, before the response ever leaves the agent loop.

The blueprint shows the minimum viable governance posture for a single-model call: one structural check on the way out, durable retries on the way in, and a full audit trail in the entity.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a prompt into the **Prompt** textarea (or picks one of three seeded examples — a summarisation request, an extraction request, and a classification request).
2. The user selects a **category hint** from a dropdown (SUMMARISE / EXTRACT / CLASSIFY / AUTO) or leaves it on AUTO to let the agent decide.
3. The user clicks **Run prompt**. The UI POSTs to `/api/runs` and receives a `runId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s the workflow starts and the card transitions to `PROCESSING`.
5. Within ~10–30 s the `processStep` completes. The card transitions to `COMPLETED`. The result appears: a category badge, a confidence score bar (0–100 %), and the output text.
6. If the agent's first response fails guardrail validation, the card stays in `PROCESSING` during the retry. The guardrail rejection never reaches the entity — the UI never shows a malformed result.
7. The user can submit another prompt; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PromptEndpoint` | `HttpEndpoint` | `/api/runs/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `PromptEntity`, `PromptView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `PromptEntity` | `EventSourcedEntity` | Per-run lifecycle: submitted → processing → completed / failed. Source of truth. | `PromptEndpoint`, `PromptWorkflow` | `PromptView` |
| `PromptWorkflow` | `Workflow` | One workflow per run. Steps: `processStep` → (on error) `errorStep`. | started by `PromptEndpoint` after submit | `PromptAgent`, `PromptEntity` |
| `PromptAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the prompt text and category hint as task instructions; returns `PromptResult`. | invoked by `PromptWorkflow` | returns result |
| `OutputGuardrail` | supporting class | Registered on `PromptAgent` via the after-agent-response hook. Validates `outputText`, `confidenceScore`, and `category` before the result leaves the agent loop. | | |
| `PromptView` | `View` | Read model: one row per run for the UI. | `PromptEntity` events | `PromptEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record PromptRequest(
    String runId,
    String promptText,
    CategoryHint categoryHint,
    String submittedBy,
    Instant submittedAt
) {}
enum CategoryHint { SUMMARISE, EXTRACT, CLASSIFY, AUTO }

record PromptResult(
    String outputText,
    double confidenceScore,   // 0.0 .. 1.0
    ResultCategory category,
    Instant producedAt
) {}
enum ResultCategory { SUMMARISE, EXTRACT, CLASSIFY }

record Run(
    String runId,
    Optional<PromptRequest> request,
    Optional<PromptResult>  result,
    RunStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RunStatus {
    SUBMITTED, PROCESSING, COMPLETED, FAILED
}
```

Events on `PromptEntity`: `RunSubmitted`, `RunStarted`, `ResultRecorded`, `RunFailed`.

Every nullable lifecycle field on the `Run` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/runs` — body `{ promptText, categoryHint, submittedBy }` → `{ runId }`.
- `GET /api/runs` — list all runs, newest-first.
- `GET /api/runs/{id}` — one run.
- `GET /api/runs/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Single-Prompt Workflow</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted runs (status pill + category badge + age) and a right pane with the selected run's detail — prompt text, category hint, result output text, confidence score bar, and category badge.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — after-agent-response guardrail**: runs on every turn of `PromptAgent`. Asserts the candidate response is a well-formed `PromptResult`, that `outputText` is non-empty, that `confidenceScore` is in the range `[0.0, 1.0]`, and that `category` is one of `{SUMMARISE, EXTRACT, CLASSIFY}`. On any failure, returns a structured rejection naming the failed check; the agent loop consumes one iteration and retries. Passing responses flow through to `PromptWorkflow.processStep`.

## 9. Agent prompts

- `PromptAgent` → `prompts/prompt-agent.md`. The single decision-making LLM. System prompt instructs it to process the user's prompt according to the detected or supplied category hint and return a `PromptResult` with a non-empty `outputText`, a calibrated `confidenceScore`, and the correct `category` enum value.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the summarisation seed prompt; within 30 s the result appears with a non-empty `outputText`, a `confidenceScore` in `[0.0, 1.0]`, and category `SUMMARISE`.
2. **J2** — The agent's first response on a run is intentionally malformed (mock LLM path) — the `after-agent-response` guardrail rejects it; the second iteration produces a valid result; the UI never displays the malformed response.
3. **J3** — A result with `confidenceScore = 1.5` is caught by the guardrail and never reaches the entity; the log shows a `guardrail.reject` entry naming the `confidenceScore` check.
4. **J4** — Two runs submitted concurrently reach `COMPLETED` independently without cross-contamination of their entity state or workflow context.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named akka-basic demonstrating the single-agent × general cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-akka-basic. Java package io.akka.samples.singlepromptworkflow.
Akka 3.6.0. HTTP port 9383.

Components to wire (exactly):

- 1 AutonomousAgent PromptAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/prompt-agent.md>) and
  .capability(TaskAcceptance.of(PROCESS_PROMPT).maxIterationsPerTask(3)). The task
  receives the promptText and categoryHint in the task's instruction text. Output:
  PromptResult{outputText: String, confidenceScore: double (0.0..1.0),
  category: ResultCategory, producedAt: Instant}. The agent is configured with an
  after-agent-response guardrail (see G1 in eval-matrix.yaml) registered via the
  agent's guardrail-configuration block. On guardrail rejection the agent loop retries
  the response within its 3-iteration budget.

- 1 Workflow PromptWorkflow per runId with two steps:
  * processStep — emits RunStarted, then calls
    componentClient.forAutonomousAgent(PromptAgent.class, "agent-" + runId)
      .runSingleTask(TaskDef.instructions(formatPrompt(run.request)))
    — returns a taskId, then forTask(taskId).result(PROCESS_PROMPT) to fetch the result.
    On success calls PromptEntity.recordResult(result). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(2).failoverTo(PromptWorkflow::error).
  * error step — calls PromptEntity.fail(reason) and transitions to terminal. stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity PromptEntity (one per runId). State Run{runId: String,
  request: Optional<PromptRequest>, result: Optional<PromptResult>, status: RunStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. RunStatus enum: SUBMITTED,
  PROCESSING, COMPLETED, FAILED. Events: RunSubmitted{request}, RunStarted{},
  ResultRecorded{result}, RunFailed{reason}. Commands: submit, markProcessing,
  recordResult, fail, getRun. emptyState() returns Run.initial("") with no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty()
  in initial state and Optional.of(...) inside the event-applier.

- 1 View PromptView with row type RunRow (mirrors Run). Table updater consumes
  PromptEntity events. ONE query getAllRuns: SELECT * AS runs FROM prompt_view. No WHERE
  status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * PromptEndpoint at /api with POST /runs (body {promptText, categoryHint, submittedBy};
    mints runId; calls PromptEntity.submit; starts PromptWorkflow; returns {runId}),
    GET /runs (list from getAllRuns, sorted newest-first), GET /runs/{id} (one row),
    GET /runs/sse (Server-Sent Events forwarded from the view's stream-updates), and
    three /api/metadata/* endpoints serving YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* ->
    static-resources/*.

Companion files:

- PromptTasks.java declaring one Task<R> constant: PROCESS_PROMPT = Task
    .name("Process prompt")
    .description("Read the submitted prompt and return a PromptResult with outputText,
      confidenceScore, and category")
    .resultConformsTo(PromptResult.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records PromptRequest, PromptResult, Run, CategoryHint, ResultCategory,
  RunStatus.

- OutputGuardrail.java implementing the after-agent-response hook. Reads the candidate
  PromptResult from the LLM response, runs the three checks listed in eval-matrix.yaml
  G1, and either passes the response through or returns Guardrail.reject(<structured
  error>) to force the agent loop to retry.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9383 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The
  PromptAgent.definition() binds the configured provider via
  .modelProvider("${akka.javasdk.agent.default}") or the per-agent override pattern
  from the akka-context docs.

- src/main/resources/sample-events/seed-prompts.jsonl with 3 seeded prompt entries:
  a summarisation prompt (a 300-word paragraph to summarise), an extraction prompt
  (a 200-word product description from which to extract key attributes), and a
  classification prompt (a short customer message to label as BILLING / TECHNICAL /
  GENERAL). Each entry has promptText, categoryHint, and submittedBy.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies
  of the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with operations.agent_count = 1,
  agent_pattern = single-agent, decisions.authority_level = recommend-only (the output
  is advisory), oversight.human_in_loop = true, failure.failure_modes including
  "empty-output", "out-of-range-confidence", "wrong-category", "hallucinated-content";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/prompt-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Single-Prompt Workflow",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms
  section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of run cards; right = selected-run detail with prompt text, category
  hint, result output, confidence bar, and category badge). Browser title exactly:
  <title>Akka Sample: Single-Prompt Workflow</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        shape-correct outputs per agent (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf;
        /akka:build forwards the value from the Claude session env to the JVM via the
        MCP tool's environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using
        the matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed
        to the JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured
  key reference if it does not resolve at runtime. The message must not echo any
  captured key material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-
  task dispatch on the Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json, picks one entry pseudo-randomly
  per call (seedFor(runId)), and deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    process-prompt.json — 6 PromptResult entries covering all three ResultCategory
      values (2 SUMMARISE, 2 EXTRACT, 2 CLASSIFY). Each entry has non-empty
      outputText (20–60 words), confidenceScore between 0.60 and 0.95, and the
      matching category. Plus 2 deliberately MALFORMED entries: one with
      confidenceScore = 1.5 (out of range) and one with outputText = "" (empty).
      The guardrail blocks both, exercising the retry path. The mock should select a
      malformed entry on the FIRST iteration of every 3rd run (modulo seed) so J2
      and J3 are reproducible.
- A MockModelProvider.seedFor(runId) helper makes per-run selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. PromptAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion PromptTasks.java MUST
  exist.
- Lesson 4: the processStep has an explicit stepTimeout of 60s; the error step has 5s.
- Lesson 6: every nullable lifecycle field on the Run row record is Optional<T>. The
  view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: PromptTasks.java with PROCESS_PROMPT = Task.name(...).description(...)
  .resultConformsTo(PromptResult.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9383 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in
  narrative, marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS
  overrides (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only
  the reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by
  NodeList index. The DOM contains exactly five <section class="tab-panel"> elements —
  Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (PromptAgent). The
  after-response guardrail is a validation class (OutputGuardrail.java) and does NOT
  make an LLM call — keeping the pattern's "one agent" promise honest.
- The after-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the workflow step completes.
  Lesson 1's AutonomousAgent contract is the authoritative reference for how the hook
  is registered.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
  Per Lesson 25, /akka:specify handles the key during generation.
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
