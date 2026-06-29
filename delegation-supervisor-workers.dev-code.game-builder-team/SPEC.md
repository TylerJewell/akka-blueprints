# SPEC — game-builder-team

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Game Builder Team.
**One-line pitch:** Type a game idea; a director delegates the design to a game designer and the code to a code writer, runs the result through a sandbox-guarded test gate, and returns a playable game spec plus its code.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a supervisor agent decomposes a game request, delegates the design and the code to two worker agents, then assembles their outputs into one deliverable inside a Workflow. The blueprint also demonstrates two governance mechanisms — a **before-tool-call guardrail** that sandboxes the simulated code-execution tool so generated code cannot reach forbidden APIs, and a **test-gate** that blocks delivery until the generated game code passes deterministic checks.

## 3. User-facing flows

The user opens the App UI tab and submits a game idea via the form.

1. The system creates a `GameProject` in `QUEUED` and starts a `GameBuildWorkflow`.
2. `designStep` — the `GameDirector` delegates the idea to the `GameDesigner`, which returns a typed `GameSpec`. Status moves to `DESIGNING`, then the spec is recorded.
3. `codeStep` — the `GameDirector` delegates the spec to the `CodeWriter`, which returns typed `GameCode`. Status moves to `CODING`.
4. `testStep` — before the simulated run tool executes, a before-tool-call guardrail inspects the code for forbidden APIs. If it trips, the build moves to `BLOCKED`. Otherwise the test gate runs deterministic checks. Pass moves toward delivery; fail moves to `TEST_FAILED` after the retry budget is spent.
5. `assembleStep` — the `GameDirector` assembles the spec and code into the final deliverable. Status moves to `DELIVERED`.

A `RequestSimulator` (TimedAction) drips a sample game idea every 60 seconds so the App UI is not empty on first load.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `GameDirector` | `AutonomousAgent` | Supervisor: decomposes the request, delegates to the workers, assembles the deliverable, runs the sandbox guardrail before the run tool. | `GameBuildWorkflow` | returns typed results to workflow |
| `GameDesigner` | `AutonomousAgent` | Worker: turns the idea into a `GameSpec` (title, genre, mechanics, controls, win condition). | `GameBuildWorkflow` | — |
| `CodeWriter` | `AutonomousAgent` | Worker: turns the spec into `GameCode` (a single self-contained HTML/JS game). | `GameBuildWorkflow` | — |
| `GameBuildWorkflow` | `Workflow` | Orchestrates designStep → codeStep → testStep → assembleStep; owns the test-gate and retry budget. | `GameEndpoint`, `BuildRequestConsumer` | `GameProjectEntity` |
| `GameProjectEntity` | `EventSourcedEntity` | Holds one build's lifecycle (queued → designing → coding → testing → delivered / test-failed / blocked). | `GameBuildWorkflow` | `GameView` |
| `BuildRequestQueue` | `EventSourcedEntity` | Logs each submitted game idea for replay and audit. | `GameEndpoint`, `RequestSimulator` | `BuildRequestConsumer` |
| `GameView` | `View` | List-of-projects read model. | `GameProjectEntity` events | `GameEndpoint` |
| `BuildRequestConsumer` | `Consumer` | Listens to `BuildRequestQueue` events; starts a workflow per submission. | `BuildRequestQueue` events | `GameBuildWorkflow` |
| `RequestSimulator` | `TimedAction` | Drips a sample game idea every 60 s. | scheduler | `BuildRequestQueue` |
| `GameEndpoint` | `HttpEndpoint` | `/api/games/*` — submit, get, list, SSE. | — | `GameView`, `BuildRequestQueue`, `GameProjectEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6). Full detail in `reference/data-model.md`.

### Records

```java
record GameSpec(String title, String genre, List<String> mechanics, String controls, String winCondition) {}
record GameCode(String html, String entryPoint, List<String> filesTouched) {}
record TestReport(boolean passed, List<String> checks, List<String> failures) {}

record GameProject(
  String id,
  String idea,
  GameStatus status,
  Optional<Instant> queuedAt,
  Optional<Instant> designedAt,
  Optional<GameSpec> spec,
  Optional<Instant> codedAt,
  Optional<GameCode> code,
  Optional<Instant> testedAt,
  Optional<TestReport> testReport,
  Optional<Integer> testAttempts,
  Optional<Instant> deliveredAt,
  Optional<String> blockReason
) {}
```

### Status enum

```java
enum GameStatus { QUEUED, DESIGNING, CODING, TESTING, TEST_FAILED, DELIVERED, BLOCKED }
```

### Events

`BuildRequested`, `SpecDesigned`, `CodeWritten`, `TestsPassed`, `TestsFailed`, `GameDelivered`, `BuildBlocked`. Triggers are listed in `reference/data-model.md`.

## 6. API contract

Full surface, payloads, and SSE event format in `reference/api-contract.md`. Top-level surface:

```
POST /api/games                 -> { projectId }
GET  /api/games ?status=...     -> { projects: [GameProject, ...] }
GET  /api/games/{projectId}     -> GameProject
GET  /api/games/sse             -> Server-Sent Events of GameProject

GET  /api/metadata/eval-matrix  -> text/yaml
GET  /api/metadata/risk-survey  -> text/yaml
GET  /api/metadata/readme       -> text/markdown

GET  /                          -> 302 /app/index.html
GET  /app/{*path}               -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown and YAML are acceptable. Browser title: `<title>Akka Sample: Game Builder Team</title>`. Full tab description in `reference/ui-mockup.md`.

Five tabs:

1. **Overview** — eyebrow + headline + four cards (Try it / How it works / Components / API contract).
2. **Architecture** — the four mermaid diagrams from `PLAN.md`. Apply the Lesson 24 mermaid CSS overrides (state-label colour, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`).
3. **Risk Survey** — renders `/api/metadata/risk-survey` in `matrix-card` / `matrix-row` style; `TO_BE_COMPLETED_BY_DEPLOYER` values render muted.
4. **Eval Matrix** — renders `/api/metadata/eval-matrix` in the same style; the label column carries a colored mechanism pill (`guardrail` red, `ci-gate` pale yellow).
5. **App UI** — submit a game idea; live SSE list of projects; per-project status, spec, and a "Play" preview that loads the delivered `GameCode.html` into a sandboxed `<iframe srcdoc>`.

Tab switching MUST match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Delete removed panels from the DOM — never `display:none` a zombie panel.

## 8. Governance

Controls live in `eval-matrix.yaml`; the deployer survey in `risk-survey.yaml`. Two mechanisms the generated system must wire:

- **G1 — sandbox guardrail (before-tool-call):** before `testStep` invokes the simulated run tool, a before-tool-call guardrail on `GameDirector` rejects `GameCode` that references forbidden APIs (network, filesystem, `eval`, dynamic import). A trip moves the project to `BLOCKED` with a reason.
- **C1 — test gate (ci-gate · test-gate):** `testStep` runs deterministic checks against the generated code. The project cannot reach `DELIVERED` until the checks pass. Failure consumes the retry budget, then moves to `TEST_FAILED`.

## 9. Agent prompts

One system prompt per agent under `prompts/`:

- `prompts/game-director.md` — supervisor that decomposes the idea, delegates, and assembles.
- `prompts/game-designer.md` — worker that returns a `GameSpec`.
- `prompts/code-writer.md` — worker that returns a self-contained `GameCode`.

## 10. Acceptance

Full journeys in `reference/user-journeys.md`. The three or four that define "generated correctly":

1. **Submit and deliver.** POST a game idea; within ~60 s the project reaches `DELIVERED` with a non-empty `spec` and `code`, and the App UI Play preview renders.
2. **Sandbox block.** A build whose generated code references a forbidden API moves to `BLOCKED` with a `blockReason`; no run tool executes.
3. **Test-gate fail.** A build whose code fails the deterministic checks retries up to the budget, then settles in `TEST_FAILED` with a populated `testReport.failures`.
4. **Simulator seed.** With no UI interaction, `RequestSimulator` enqueues an idea within 60 s and a workflow starts.

## 11. Implementation directives

```
Create a sample named game-builder-team demonstrating the
delegation-supervisor-workers x dev-code cell. Runs out of the box (no external
services). Maven group io.akka.samples. Maven artifact game-builder-team. Java
package io.akka.samples.gamebuilderteam. Akka 3.6.0. HTTP port 9489.

Matrix cell: coordination_pattern delegation-supervisor-workers, domain dev-code.
Integration: runs out of the box. The code-execution / run surface is simulated
in-process — a deterministic SandboxRunner helper invoked from testStep, NOT a
browser, Docker, or external sandbox.

Components to wire (exactly):
- 3 AutonomousAgents:
  - GameDirector — supervisor. definition() with
    capability(TaskAcceptance.of(task).maxIterationsPerTask(3)). Delegates by
    returning typed work items; assembles GameSpec + GameCode into the
    deliverable. Hosts the before-tool-call guardrail that vets GameCode before
    the run tool fires.
  - GameDesigner — worker. Returns a typed GameSpec.
  - CodeWriter — worker. Returns a typed GameCode (single self-contained HTML/JS).
  Each declares definition() with capability(TaskAcceptance.of(task)
  .maxIterationsPerTask(3)).
- 1 Workflow GameBuildWorkflow with steps designStep -> codeStep -> testStep ->
  assembleStep. designStep and codeStep call forAutonomousAgent(...).runSingleTask
  (...) then forTask(taskId).result(...). testStep: (1) run the before-tool-call
  guardrail on the GameCode; if it trips, call GameProjectEntity.block(reason) and
  end; (2) run SandboxRunner deterministic checks; on pass call recordTestsPassed
  and proceed to assembleStep; on fail increment attempts, retry codeStep up to 2
  times, then call recordTestsFailed and end. Override settings() with
  stepTimeout(60s) on designStep, codeStep, assembleStep and
  defaultStepRecovery(maxRetries(2).failoverTo(error)).
- 1 EventSourcedEntity GameProjectEntity holding a GameProject record with id,
  idea, GameStatus enum, and Optional lifecycle fields (queuedAt, designedAt,
  spec, codedAt, code, testedAt, testReport, testAttempts, deliveredAt,
  blockReason). Events: BuildRequested, SpecDesigned, CodeWritten, TestsPassed,
  TestsFailed, GameDelivered, BuildBlocked. Commands: start, recordSpec,
  recordCode, recordTestsPassed, recordTestsFailed, deliver, block, getProject.
  emptyState() returns GameProject.initial("", "") with no commandContext()
  reference.
- 1 EventSourcedEntity BuildRequestQueue with a single command enqueue(idea)
  emitting BuildEnqueued.
- 1 View GameView with row type GameProject, table updater consuming
  GameProjectEntity events. ONE query: getAllProjects SELECT * AS projects FROM
  game_view. No WHERE status filter (Akka cannot auto-index enum columns) —
  filter client-side in callers.
- 1 Consumer BuildRequestConsumer subscribed to BuildRequestQueue events; on each
  event starts a GameBuildWorkflow with a fresh UUID.
- 1 TimedAction RequestSimulator (every 60s, reads the next line from
  src/main/resources/sample-events/game-ideas.jsonl and calls
  BuildRequestQueue.enqueue).
- 2 HttpEndpoints: GameEndpoint at /api/games with submit, get, list (filter
  client-side from getAllProjects), SSE stream, and three /api/metadata/*
  endpoints serving the YAML/MD files from src/main/resources/metadata/.
  AppEndpoint serving / -> 302 /app/index.html and /app/* ->
  static-resources/*.

Companion files:
- GameBuildTasks.java declaring Task<R> constants: DESIGN (resultConformsTo
  GameSpec), CODE (GameCode), ASSEMBLE (GameProject or a Deliverable record).
- Records GameSpec(title, genre, mechanics, controls, winCondition),
  GameCode(html, entryPoint, filesTouched), TestReport(passed, checks, failures).
- SandboxRunner.java — a deterministic in-process helper: a forbidden-API scan
  (network, filesystem, eval, dynamic import) used by the guardrail, plus smoke
  checks (HTML present, entry point referenced, no syntax markers) used by the
  test gate. No external process.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9489
  and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each
  api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/game-ideas.jsonl with 8 canned game-idea lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with controls G1, C1 and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling the inferable fields and
  marking deployer-specific fields TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root (already authored): pitch, prerequisites,
  generate-the-system block ending in /akka:specify @SPEC.md, what-you-get,
  customise, what-gets-validated, license. No governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained HTML
  file (no ui/ folder, no npm build). Five tabs (Overview, Architecture, Risk
  Survey, Eval Matrix, App UI). Match the governance.html visual style
  (dark / yellow / Instrument Sans / dot-grid). App UI Play preview loads
  delivered GameCode.html into a sandboxed <iframe srcdoc> (sandbox attribute,
  no allow-same-origin).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, inspect the environment for ANTHROPIC_API_KEY /
  OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set, default
  application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (GameDirector -> deliverable assembly, GameDesigner -> GameSpec,
  CodeWriter -> GameCode; see src/main/resources/mock-responses/
  {game-director,game-designer,code-writer}.json with 4-6 entries each). Sets
  model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed
  to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured reference
  if it does not resolve at runtime; never echoes any captured key.

Mock LLM provider — per-agent mock-response shapes:
- GameDesigner -> GameSpec{title, genre, mechanics[2-4], controls, winCondition}.
- CodeWriter -> GameCode{html (a minimal self-contained canvas game that passes
  the sandbox scan and smoke checks), entryPoint, filesTouched}.
- GameDirector -> the assembled deliverable referencing the spec and code ids.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Lesson 1: AutonomousAgent must extend AutonomousAgent — never silently
  downgraded to Agent.
- Lesson 4: every workflow step that calls an agent overrides stepTimeout (60s).
- Lesson 6: Optional<T> for every nullable lifecycle field on the GameProject
  row record.
- Lesson 7: AutonomousAgent requires the companion GameBuildTasks.java.
- Lesson 8: verify model names are current before locking them in
  application.conf.
- Lesson 9: the run command is "/akka:build", never "mvn akka:run".
- Lesson 10: explicit dev-mode http-port = 9489 in application.conf.
- Lesson 11: never render source-platform metadata anywhere user-facing.
- Lesson 12: UI fits the content column with no horizontal scroll.
- Lesson 13: integration label is "Runs out of the box" — never T1/T2/T3/T4.
- Lesson 23: no competitor brand names anywhere user-facing.
- Lesson 24: include the mermaid state-label CSS overrides and theme variables.
- Lesson 25: five-option key sourcing; never write the key to disk.
- Lesson 26: tab switching by data-tab / data-panel attribute, no zombie panels.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification, **do not stop and wait for the user**. Continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and 6.
2. Run `/akka:tasks` — break the plan into tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL (`http://localhost:9489/`) and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — a missing API key (offer the three valid env vars or the mock option), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
