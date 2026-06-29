# SPEC — code-team-team

The natural-language brief `/akka:specify @SPEC.md` reads to generate this system. Sections 1–11 together are the input; Section 12 chains the rest of the workflow.

---

## 1. System name + pitch

**System name:** Game Builder Team.
**One-line pitch:** The user types a one-line game brief (e.g. "a number-guessing CLI game"); a senior-engineer agent writes the Python game, a QA agent runs tests against it, and a chief-QA agent decides ship or rework. The reviewed source and the test outcome stream back to the UI.

## 2. What this blueprint demonstrates

A sequential pipeline: three agents run in a fixed order — engineer, QA, chief QA — each consuming the prior stage's typed output. The governance pattern wires three controls: a before-tool-call guardrail around the in-process code-execution sandbox the QA stage uses, an in-band evaluation event that records the QA score on each build, and a test-gate that prevents a build from reaching SHIPPED unless its automated tests pass.

## 3. User-facing flows

1. The user submits a game brief through the App UI (or `POST /api/build-request`). The response carries a `buildId` and the build appears in QUEUED.
2. The engineer agent writes the game; the build moves to ENGINEERED and the source code is shown.
3. The QA agent runs tests in the guarded sandbox; the build records a QA score and moves to QA. The eval score appears beside the build.
4. The chief-QA agent reviews. If QA passed and the chief signs off, the build moves to SHIPPED. Otherwise it moves to REJECTED with a rework reason.
5. Without any interaction, the `BriefSimulator` seeds new briefs from a JSONL file every 30 seconds, so the pipeline shows activity on its own.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| EngineerAgent | AutonomousAgent | Writes the Python game from the brief; returns `GameCode{filename, source}` | GameBuildWorkflow | BuildEntity |
| QaAgent | AutonomousAgent | Runs tests via the guarded sandbox tool; returns `QaReport{passed, score, notes}` | GameBuildWorkflow | BuildEntity |
| ChiefQaAgent | AutonomousAgent | Final ship/rework call; returns `ShipDecision{shipped, summary}` | GameBuildWorkflow | BuildEntity |
| GameBuildWorkflow | Workflow | Runs engineerStep → qaStep → chiefReviewStep in sequence | BriefConsumer | EngineerAgent, QaAgent, ChiefQaAgent, BuildEntity |
| BuildEntity | EventSourcedEntity | Holds one build's lifecycle and emits events | GameBuildWorkflow | BuildsView |
| InboundBriefQueue | EventSourcedEntity | Records inbound briefs as events | BriefSimulator, BuildEndpoint | BriefConsumer |
| BuildsView | View | CQRS read model over BuildEntity events | BuildEntity | BuildEndpoint |
| BriefConsumer | Consumer | Starts one GameBuildWorkflow per queued brief | InboundBriefQueue | GameBuildWorkflow |
| BriefSimulator | TimedAction | Drips canned briefs into the queue every 30s | (timer) | InboundBriefQueue |
| BuildEndpoint | HttpEndpoint | REST + SSE for builds and metadata | UI | InboundBriefQueue, BuildsView |
| AppEndpoint | HttpEndpoint | Serves the static UI | browser | static-resources |

Names are used verbatim by `/akka:specify`.

## 5. Data model

See `reference/data-model.md` for the full field tables.

`Build` record (BuildEntity state and BuildsView row): `id` (String), `brief` (`Optional<String>`), `status` (`BuildStatus`), and the nullable lifecycle fields `engineeredAt`, `filename`, `sourceCode`, `qaAt`, `qaPassed` (`Optional<Boolean>`), `qaScore` (`Optional<Integer>`), `qaNotes`, `chiefReviewedAt`, `shipped` (`Optional<Boolean>`), `shipSummary`, `reworkReason` — every one declared `Optional<T>` (Lesson 6).

`BuildStatus` enum: `QUEUED`, `ENGINEERED`, `QA`, `SHIPPED`, `REJECTED`.

Events on BuildEntity: `GameEngineered`, `QaCompleted`, `BuildShipped`, `BuildRejected`. Event on InboundBriefQueue: `BriefQueued`.

Typed agent results: `GameCode(String filename, String source)`, `QaReport(boolean passed, int score, String notes)`, `ShipDecision(boolean shipped, String summary)`.

## 6. API contract

Top-level surface (schemas in `reference/api-contract.md`):

```
POST /api/build-request            -> { buildId }
GET  /api/builds                   -> { builds: [Build, ...] }
GET  /api/builds/{buildId}         -> Build
GET  /api/builds/sse               -> Server-Sent Events of Build
GET  /api/metadata/eval-matrix     -> text/yaml
GET  /api/metadata/risk-survey     -> text/yaml
GET  /api/metadata/readme          -> text/markdown
GET  /                             -> 302 /app/index.html
GET  /app/{*path}                  -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` — no `ui/` folder, no npm build. Browser title: `<title>Akka Sample: Game Builder Team</title>`. Five tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index, with no hidden zombie panels (Lesson 26). Mermaid diagrams on the Architecture tab carry the state-label CSS overrides and theme variables from Lesson 24. The App UI tab submits a brief and streams the live build list over SSE. Full description: `reference/ui-mockup.md`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. Three controls:

- **G1 (guardrail · before-tool-call):** the QA sandbox tool is wrapped by a before-tool-call guardrail that rejects source containing forbidden operations (network, filesystem escape, `os.system`/`subprocess`) before any simulated execution.
- **E1 (eval-event · on-decision-eval):** when the QA stage completes, an evaluation event records a quality score on the build; the UI shows it beside the source.
- **A1 (ci-gate · test-gate):** a build cannot reach SHIPPED unless its automated tests pass; the chief-review step enforces the gate, and the Maven test phase enforces it at build time.

## 9. Agent prompts

- `prompts/engineer-agent.md` — writes a small self-contained Python game from the brief.
- `prompts/qa-agent.md` — writes and runs tests against the game and scores it.
- `prompts/chief-qa-agent.md` — makes the final ship-or-rework decision.

## 10. Acceptance

Inlined from `reference/user-journeys.md`:

1. **Submit a brief and watch it ship.** A brief that the team handles cleanly moves QUEUED → ENGINEERED → QA → SHIPPED with non-empty `sourceCode` and a `qaScore`.
2. **A failing build is rejected, never shipped.** A brief whose tests fail lands in REJECTED with a `reworkReason`; `shipped` is never true.
3. **Sandbox guardrail blocks unsafe code.** Source containing a forbidden operation is rejected by the before-tool-call guardrail; the build does not reach SHIPPED.
4. **Background load.** The simulator seeds briefs every 30 s; builds appear without any UI action.

---

## 11. Implementation directives

```
Create a sample named code-team-team demonstrating the sequential-pipeline ×
dev-code cell. Runs out of the box (no external services; the code-execution
sandbox is modeled in-process). Maven group io.akka.samples. Maven artifact
code-team-team. Java package io.akka.samples.codeteamteam. Akka 3.6.0. HTTP
port 9375.

Components to wire (exactly):
- 3 AutonomousAgents:
  - EngineerAgent — returns typed GameCode{filename, source}. definition()
    with capability(TaskAcceptance.of(BuildTasks.ENGINEER).maxIterationsPerTask(3)).
  - QaAgent — returns typed QaReport{passed, score, notes}. Uses a guarded
    sandbox tool (CodeRunner) to "run" tests. capability(...maxIterationsPerTask(3)).
  - ChiefQaAgent — returns typed ShipDecision{shipped, summary}.
    capability(...maxIterationsPerTask(2)).
- 1 Workflow GameBuildWorkflow with steps engineerStep -> qaStep ->
  chiefReviewStep. Each step calls forAutonomousAgent(Agent.class,...)
  .runSingleTask(...) then forTask(taskId).result(...), then writes the
  matching event to BuildEntity. chiefReviewStep ships only if QaReport.passed
  AND ShipDecision.shipped; otherwise calls BuildEntity.reject. Override
  settings() with stepTimeout(60s) on engineerStep, qaStep, chiefReviewStep
  and defaultStepRecovery(maxRetries(2).failoverTo(error)). WorkflowSettings is
  nested inside Workflow — no import.
- 1 EventSourcedEntity BuildEntity holding a Build record with id, brief
  (Optional<String>), BuildStatus enum, and 11 Optional lifecycle fields
  (engineeredAt, filename, sourceCode, qaAt, qaPassed Optional<Boolean>,
  qaScore Optional<Integer>, qaNotes, chiefReviewedAt, shipped
  Optional<Boolean>, shipSummary, reworkReason). Events: GameEngineered,
  QaCompleted, BuildShipped, BuildRejected. Commands: recordEngineered,
  recordQa, ship, reject, getBuild. emptyState() returns Build.initial("")
  with no commandContext() reference.
- 1 EventSourcedEntity InboundBriefQueue with a single command
  enqueueBrief(brief) emitting BriefQueued.
- 1 View BuildsView with row type Build, table updater consuming BuildEntity
  events. ONE query: getAllBuilds SELECT * AS builds FROM builds_view. No
  WHERE status filter (Akka cannot auto-index enum columns) — filter
  client-side in callers.
- 1 Consumer BriefConsumer subscribed to InboundBriefQueue events; on each
  event starts a GameBuildWorkflow with a fresh UUID.
- 1 TimedAction BriefSimulator (every 30s, reads the next line from
  src/main/resources/sample-events/game-briefs.jsonl and calls
  InboundBriefQueue.enqueueBrief).
- 2 HttpEndpoints: BuildEndpoint at /api with build-request, builds list
  (filter client-side from getAllBuilds), single build, SSE stream, and three
  /api/metadata/* endpoints serving the YAML/MD files from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html
  and /app/* -> static-resources/*.

Companion files:
- BuildTasks.java declaring three Task<R> constants: ENGINEER (resultConformsTo
  GameCode), QA (QaReport), CHIEF (ShipDecision).
- GameCode(String filename, String source), QaReport(boolean passed, int score,
  String notes), ShipDecision(boolean shipped, String summary).
- A guarded CodeRunner tool used by QaAgent. The before-tool-call guardrail
  inspects the candidate source and rejects forbidden operations (network,
  filesystem escape, os.system/subprocess/eval) before the simulated run.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9375 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. Verify each model name is current before
  locking it.
- src/main/resources/sample-events/game-briefs.jsonl with 8 canned brief lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 3 controls (G1, E1, A1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions,
  data.types, capability.*, model.*, subjects.children; marking jurisdictions,
  declared_frameworks, organization fields, incidents, and data.residency as
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root: pitch, component inventory, matrix cell,
  integration descriptor, how to run, the tabs, an ASCII architecture diagram,
  project layout, API contract, license. NO governance-mechanisms section. NO
  configuration section.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN imports
  for markdown and YAML libs are acceptable. Five tabs: Overview, Architecture,
  Risk Survey, Eval Matrix, App UI. Match the governance.html visual style
  (dark/yellow/Instrument Sans/dot-grid).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify inspects the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (EngineerAgent -> GameCode, QaAgent -> QaReport, ChiefQaAgent ->
  ShipDecision; see src/main/resources/mock-responses/{engineer-agent,
  qa-agent,chief-qa-agent}.json with 4-6 entries each). Sets model-provider =
  mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed
  to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured
  reference if it does not resolve at runtime; never echoes any captured key.

Mock LLM provider per-agent shapes (option a):
- EngineerAgent -> GameCode{filename: "game.py", source: a short syntactically
  valid Python game (~15-40 lines)}.
- QaAgent -> QaReport{passed: mostly true, score: 60-100, notes: one line}.
- ChiefQaAgent -> ShipDecision{shipped: true when QaReport.passed, summary:
  one line}.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause
  matches the spec verbatim.
- Lesson 4: every agent-calling workflow step sets an explicit stepTimeout(60s).
- Lesson 6: Optional<T> for every nullable lifecycle field on the Build row
  record.
- Lesson 7: AutonomousAgents require the companion BuildTasks.java.
- Lesson 8: verify model names are current before locking them.
- Lesson 9: run command is "/akka:build", never "mvn akka:run".
- Lesson 10: explicit dev-mode http-port 9375 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column, no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never
  T1/T2/T3/T4 or "deferred".
- Lesson 23: no competitor brand names anywhere user-facing.
- Lesson 24: static-resources/index.html includes the mermaid state-diagram
  label CSS overrides and theme variables (edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc).
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching by data-tab / data-panel attribute, never NodeList
  index; no hidden zombie panels — delete removed tabs, do not display:none
  them.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and the PLAN.md.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — an unresolved API key reference (offer the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
