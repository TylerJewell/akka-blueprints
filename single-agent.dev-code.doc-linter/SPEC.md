# SPEC — doc-linter

The natural-language brief `/akka:specify @SPEC.md` reads to generate this system. Sections 1–11 together are the input; Section 12 drives the rest of the workflow.

---

## 1. System name + pitch

**System name:** Doc Linter.
**One-line pitch:** The user picks a Markdown file and clicks Lint; a general-assistant agent runs a custom lint tool over the file and returns a plain-language summary of the edits worth making. The user can record feedback that tunes which rules the next summary emphasizes.

## 2. What this blueprint demonstrates

The single-agent coordination pattern: one agent owns the whole task, calling a custom in-process tool to do the deterministic work and using the model only to interpret and summarize. The governance pattern pairs a before-tool-call guardrail that confines file reads to a fixed root with a documentation gate that turns error-severity findings into a pass/fail signal a merge step can read.

## 3. User-facing flows

1. The user opens the App UI tab and selects one of the sample Markdown files, then clicks Lint. A lint run appears in `REQUESTED`, then moves to `LINTED` with a findings table and a summary once the agent completes (typically 5–30 s).
2. The user reads the summary and the gate badge (PASS when there are no error-severity findings, FAIL otherwise).
3. The user records feedback on a finding ("too strict", "keep flagging this"). The run moves to `FEEDBACK_APPLIED` and the rule profile shifts.
4. If the user enters a path that escapes the sample-data root, the run is recorded as `BLOCKED` and no file is read.
5. Without any interaction, `LintSimulator` seeds lint runs from the sample file list so the list is never empty.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| LinterAgent | Agent | Runs the custom Markdown lint tool over a file, then summarizes findings into actionable edits, weighted by the rule profile | LintEndpoint | LintEntity |
| LintEntity | EventSourcedEntity | Records a lint run lifecycle: requested, completed, feedback, blocked | LinterAgent, LintEndpoint | LintView, FeedbackConsumer |
| RuleProfileEntity | KeyValueEntity | Holds learned per-rule weights tuned by user feedback | FeedbackConsumer | LinterAgent |
| LintView | View | Projects lint runs into a read model for query and SSE | LintEntity | LintEndpoint |
| FeedbackConsumer | Consumer | On `FeedbackRecorded`, adjusts the matching rule weight in RuleProfileEntity | LintEntity | RuleProfileEntity |
| LintSimulator | TimedAction | Every 30 s, seeds a lint run from the next sample file path | sample-data | LintEndpoint, LinterAgent |
| LintEndpoint | HttpEndpoint | REST + SSE + metadata at `/api` | UI | LinterAgent, LintEntity |
| AppEndpoint | HttpEndpoint | Serves `/` → `/app/index.html` and `/app/*` static files | browser | static-resources |

The custom lint tool (`MarkdownLintTool`) is a function tool registered on `LinterAgent`, not a separate primitive. The before-tool-call guardrail (control G1) fires on its file-read step.

## 5. Data model

See `reference/data-model.md`. Records (lifecycle fields are `Optional<T>` per Lesson 6):

- `LintRun(String id, String filePath, LintStatus status, Instant requestedAt, Optional<Instant> lintedAt, List<Finding> findings, Optional<String> summary, Optional<GateStatus> gate, Optional<String> feedbackNote, Optional<Instant> feedbackAt, Optional<String> blockReason)`.
- `Finding(String rule, int line, Severity severity, String message)`.
- `RuleProfile(Map<String,Integer> weights)`.
- `LintStatus` enum: `REQUESTED, LINTED, FEEDBACK_APPLIED, BLOCKED`.
- `GateStatus` enum: `PASS, FAIL`.
- `Severity` enum: `ERROR, WARNING, INFO`.
- Events: `LintRequested`, `LintCompleted`, `FeedbackRecorded`, `LintBlocked`.

## 6. API contract

See `reference/api-contract.md` for payload schemas. Top-level surface:

```
POST /api/lint                 -> { runId }
POST /api/lint/{id}/feedback   -> 200 | 404
GET  /api/lint                 -> { runs: [LintRun, ...] }
GET  /api/lint/{id}            -> LintRun
GET  /api/lint/sse             -> Server-Sent Events of LintRun
GET  /api/files                -> { files: [path, ...] }
GET  /api/metadata/eval-matrix -> text/yaml
GET  /api/metadata/risk-survey -> text/yaml
GET  /api/metadata/readme      -> text/markdown
GET  /                         -> 302 /app/index.html
GET  /app/{*path}              -> static-resources/{*path}
```

## 7. UI

Five tabs in a single self-contained `src/main/resources/static-resources/index.html` — see `reference/ui-mockup.md`. Browser title: `<title>Akka Sample: Doc Linter</title>`.

- **Overview** — eyebrow "Overview" + headline naming the sample type, then the four cards Try it / How it works / Components / API contract.
- **Architecture** — the four mermaid diagrams from `PLAN.md`, with the Lesson 24 CSS overrides for state-diagram labels and edge labels.
- **Risk Survey** — `/api/metadata/risk-survey` in `matrix-card` / `matrix-row` style; `TO_BE_COMPLETED_BY_DEPLOYER` values muted.
- **Eval Matrix** — `/api/metadata/eval-matrix` in the same style; the label column carries a colored mechanism pill (guardrail red, ci-gate pale yellow).
- **App UI** — pick a sample file, click Lint, watch the run list over SSE, read findings + summary + gate badge, and record feedback on a finding.

Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26). No hidden zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. Two controls:

- **G1 — guardrail · before-tool-call.** Before `MarkdownLintTool` reads a file, a guardrail canonicalizes the path and blocks anything outside the sample-data root; the run is recorded `BLOCKED`.
- **A1 — ci-gate · documentation-gate.** A lint run's gate is `FAIL` when any finding has `ERROR` severity, `PASS` otherwise; an integration test asserts the gate so a merge step can read it.

## 9. Agent prompts

- `LinterAgent` → `prompts/linter-agent.md` — runs the lint tool, then writes a short, ordered summary of the edits worth making, weighted by the rule profile.

## 10. Acceptance

See `reference/user-journeys.md`. The journeys that mean "generated correctly":

1. **Lint a file.** POST `/api/lint` with a sample path returns a `runId`; the run reaches `LINTED` with a non-empty findings list and a summary.
2. **Read the gate.** A file with an error-severity issue shows gate `FAIL`; a clean file shows `PASS`.
3. **Block a traversal.** POST `/api/lint` with `../../etc/passwd` records the run `BLOCKED` with a reason and reads no file.
4. **Apply feedback.** POST feedback on a finding moves the run to `FEEDBACK_APPLIED` and changes the matching rule weight.

---

## 11. Implementation directives

```
Create a sample named doc-linter demonstrating the single-agent x dev-code
cell. Runs out of the box (no external services). Maven group io.akka.samples.
Maven artifact doc-linter. Java package io.akka.samples.doclinter. Akka 3.6.0.
HTTP port 9251.

Components to wire (exactly):
- 1 Agent LinterAgent. A request/response Agent (extends akka.javasdk.agent.Agent),
  NOT an AutonomousAgent. Entry method lint(LintCommand{runId, filePath, profile})
  returning Effect<LintResult> via effects().systemMessage(prompts/linter-agent.md)
  .userMessage(...).responseAs(LintResult.class).thenReply(). Register a function
  tool MarkdownLintTool that reads the file under the sample-data root and returns
  List<Finding> from deterministic checks (heading increment, trailing whitespace,
  hard tabs, line length, missing top-level H1, bare URL, unclosed code fence).
  The model's job is only to order and summarize findings using RuleProfile weights,
  not to invent findings.
- 1 EventSourcedEntity LintEntity holding LintRun. Commands: requestLint(filePath),
  recordResult(findings, summary, gate), recordFeedback(rule, note), markBlocked(reason),
  getRun. Events: LintRequested, LintCompleted, FeedbackRecorded, LintBlocked.
  emptyState() returns LintRun.initial("") with no commandContext() reference.
  Every nullable lifecycle field is Optional<T> (lintedAt, summary, gate, feedbackNote,
  feedbackAt, blockReason).
- 1 KeyValueEntity RuleProfileEntity holding RuleProfile{Map<String,Integer> weights}.
  Commands: bumpWeight(rule, delta), getProfile. Default weight 1 per rule.
- 1 View LintView with row type LintRun, table updater consuming LintEntity events.
  ONE query: getAllRuns SELECT * AS runs FROM lint_view. No WHERE on the status enum
  (Akka cannot auto-index enum columns) — filter client-side in callers.
- 1 Consumer FeedbackConsumer subscribed to LintEntity events; on FeedbackRecorded,
  calls RuleProfileEntity.bumpWeight for the named rule.
- 1 TimedAction LintSimulator (every 30s): reads the next path from
  src/main/resources/sample-data/manifest.txt and POSTs a lint run via the endpoint
  flow (requestLint + LinterAgent.lint). Skips paths already linted in this run set.
- 2 HttpEndpoints: LintEndpoint at /api with lint, feedback, runs list (filter
  client-side from getAllRuns), single run, SSE stream, files list, and three
  /api/metadata/* endpoints serving the YAML/MD from src/main/resources/metadata/.
  AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Guardrail wiring (control G1):
- A before-tool-call guardrail on MarkdownLintTool canonicalizes the requested path
  (resolve + normalize) and asserts it stays under the sample-data root. On violation
  it blocks the tool call; the endpoint records LintEntity.markBlocked(reason) and the
  run surfaces as BLOCKED. Never read a file before this check passes.

Documentation gate (control A1):
- gate = FAIL if any Finding.severity == ERROR, else PASS, computed in recordResult.
- An integration test drives lint over a known-bad and a known-good sample file and
  asserts FAIL then PASS, so a merge step can read the gate.

Companion files:
- Records: LintRun, Finding, RuleProfile, LintCommand, LintResult, plus the three enums
  LintStatus, GateStatus, Severity.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9251 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-data/ with manifest.txt and ~6 markdown files, a mix of
  clean files and files with seeded ERROR/WARNING/INFO issues.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (classpath
  copies of the root files the endpoint serves).
- eval-matrix.yaml at the project root with controls G1 and A1 plus simplified_view.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data, capability,
  model, oversight; marking deployer fields TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root (already authored): elevator pitch, component inventory,
  matrix cell, integration descriptor, how to run, the tabs, API contract, license.
  No governance-mechanisms section. No configuration section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file
  (no ui/ folder, no npm build). Inline CSS + JS; runtime CDN imports for markdown and
  YAML are acceptable. Five tabs per Section 7. Match the governance.html visual style
  (dark / yellow accent / Instrument Sans / dot-grid background).

Generation workflow -- see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding, /akka:specify inspects the environment for ANTHROPIC_API_KEY /
  OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set, default
  application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM -- generate a MockModelProvider with per-agent canned/random outputs
  (LinterAgent -> LintResult ordering + summary; see
  src/main/resources/mock-responses/linter-agent.json with 4-6 entries). Sets
  model-provider = mock.
  (b) Name an existing env var -- record the NAME in application.conf.
  (c) Point to an existing env file path -- record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI -- recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session -- value lives only in Claude session memory; passed to
  the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Record only the REFERENCE --
  env-var name, file path, or secrets URI -- never the value.
- Bootstrap.java fails fast with a clear message naming the configured reference if it
  does not resolve at runtime; never echoes any captured key.

Mock LLM provider (option a) per-agent shapes:
- LinterAgent: returns a LintResult{ orderedFindingRules:[...], summary:"3-6 line plain
  summary of the edits worth making" }. The findings themselves come from the
  deterministic tool, not the model; the mock only orders them and writes the summary.

Constraints -- see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- AI primitive matches the spec verbatim -- LinterAgent extends Agent, never silently
  swapped to AutonomousAgent (Lesson 1).
- Workflow step timeouts: not applicable (no Workflow here); if one is added, override
  settings() with stepTimeout >= 60s on agent-calling steps (Lesson 4).
- Optional<T> for every nullable lifecycle field on the LintRun view row (Lesson 6).
- AutonomousAgent companion Tasks.java only if an AutonomousAgent is used; not here
  (Lesson 7).
- Verify model-name strings are current before locking them (Lesson 8).
- Run command is "/akka:build" (Claude Code slash command), never "mvn akka:run"
  (Lesson 9).
- Port 9251 declared in application.conf (Lesson 10).
- Never render source.platform anywhere user-facing (Lesson 11).
- UI fits the 1080px content column with no horizontal scroll (Lesson 12).
- Integration label is the descriptive "Runs out of the box", never T1/T2/T3/T4
  (Lesson 13, 23).
- Metadata tabs render in matrix-card / matrix-row style (Lesson 18).
- static-resources/index.html includes the mermaid CSS overrides and theme variables
  from Lesson 24 (state-diagram label colour, edge-label foreignObject overflow:visible,
  transitionLabelColor #cccccc).
- Tab switching matches by data-tab / data-panel attribute, NEVER by NodeList index
  (Lesson 26). No display:none zombie panels -- delete removed tabs.
- No forbidden words anywhere (no shape, minimal, smaller, complex, "Akka SDK" in
  narrative, use, use, competitor brand names).
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and 6.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL (`http://localhost:9251/`) and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars or the mock model), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
