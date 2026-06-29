# SPEC — debate

The natural-language brief `/akka:specify @SPEC.md` reads to generate this system. Sections 1–11 together are the input; Section 12 chains the rest of the pipeline.

---

## 1. System name + pitch

**System name:** Multi-Perspective Debate (UI title: `Akka Sample: Multi-Perspective Debate`).
**One-line pitch:** The user enters a debate topic; a DebateModerator runs up to five alternating rounds between an Advocate and a Critic, then a Synthesizer returns a balanced conclusion with the key arguments from each side.

## 2. What this blueprint demonstrates

The debate-multi-perspective coordination pattern: a Workflow alternates two opposed AutonomousAgents across bounded rounds, accumulating their arguments on an event-sourced entity, then a third agent synthesises a balanced conclusion. The governance pattern is a before-agent-response guardrail on the user-facing synthesis (balance and safety) plus an on-decision-eval event that scores synthesis quality against the recorded rounds. Both are wired with first-party Akka components and run out of the box.

## 3. User-facing flows

1. The user submits a topic through the App UI (or `POST /api/debates`). The response includes the `debateId`. The debate appears in `DEBATING` state.
2. The DebateModerator runs round 1: the Advocate argues in favour, then the Critic rebuts. The round is recorded on the entity and streamed to the UI.
3. Rounds continue, alternating, until either both sides converge or `maxRounds` (5) is reached.
4. The Synthesizer produces a balanced conclusion plus a list of key arguments. The guardrail checks the synthesis for balance and safety before it is persisted.
5. A quality eval fires on the synthesis and records a score. The UI shows the conclusion, key arguments, every round, and the eval score.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `Advocate` | AutonomousAgent | Argues in favour of the topic each round | `DebateModerator` | returns `Argument` |
| `Critic` | AutonomousAgent | Argues against / rebuts each round | `DebateModerator` | returns `Argument` |
| `Synthesizer` | AutonomousAgent | Writes balanced conclusion + key arguments | `DebateModerator` | returns `Synthesis` |
| `DebateModerator` | Workflow | Orchestrates opening, rounds, synthesis | `DebateRequestConsumer` | `DebateEntity`, agents |
| `DebateEntity` | EventSourcedEntity | Holds debate state, rounds, conclusion | `DebateModerator` | `DebatesView`, `SynthesisEvalConsumer` |
| `InboundRequestQueue` | EventSourcedEntity | Queues inbound topics | `DebateEndpoint`, `DebateSimulator` | `DebateRequestConsumer` |
| `DebatesView` | View | Read model for UI list + SSE | `DebateEntity` | `DebateEndpoint` |
| `DebateRequestConsumer` | Consumer | Starts a workflow per queued topic | `InboundRequestQueue` | `DebateModerator` |
| `SynthesisEvalConsumer` | Consumer | Fires quality eval on each synthesis | `DebateEntity` | `DebateEntity` |
| `DebateSimulator` | TimedAction | Seeds canned topics every 30s | `sample-events/*.jsonl` | `InboundRequestQueue` |
| `DebateEndpoint` | HttpEndpoint | API: submit, list, single, SSE, metadata | clients | `InboundRequestQueue`, `DebatesView` |
| `AppEndpoint` | HttpEndpoint | Serves the static UI | browser | static-resources |

## 5. Data model

Authoritative — `/akka:implement` writes these records verbatim. See `reference/data-model.md`.

- `Debate` (entity state + view row): `id` String; `topic` Optional<String>; `status` DebateStatus enum; `currentRound` int; `maxRounds` int; `rounds` List<Round>; `conclusion` Optional<String>; `keyArguments` Optional<List<String>>; `qualityScore` Optional<Double>; `createdAt` Optional<Instant>; `concludedAt` Optional<Instant>; `failedReason` Optional<String>.
- `Round` (value record): `number` int; `advocateArgument` String; `criticArgument` String.
- `DebateStatus` enum: `PENDING`, `DEBATING`, `SYNTHESIZING`, `CONCLUDED`, `FAILED`.
- Events: `DebateStarted`, `RoundRecorded`, `SynthesisRecorded`, `SynthesisEvaluated`, `DebateFailed`.

Every nullable lifecycle field is `Optional<T>` (Lesson 6). The view row reuses the `Debate` record, so its nullable columns are `Optional<T>` too.

## 6. API contract

All endpoints are JSON. ACL open to the internet (local-dev only). Payload schemas and SSE format in `reference/api-contract.md`.

```
POST /api/debates                 -> { debateId }
GET  /api/debates ?status=...     -> { debates: [Debate, ...] }
GET  /api/debates/{id}            -> Debate
GET  /api/debates/sse             -> Server-Sent Events of Debate
GET  /api/metadata/eval-matrix    -> text/yaml
GET  /api/metadata/risk-survey    -> text/yaml
GET  /api/metadata/readme         -> text/markdown
GET  /                            -> 302 /app/index.html
GET  /app/{*path}                 -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown and YAML are allowed. Browser title `<title>Akka Sample: Multi-Perspective Debate</title>`. Five tabs: Overview, Architecture, Risk Survey, Eval Matrix, App UI — see `reference/ui-mockup.md`.

- Tab switching matches by `data-tab` / `data-panel` attribute, never by NodeList index; no hidden zombie panels (Lesson 26).
- The Architecture tab renders the mermaid diagrams with the Lesson 24 state-label CSS overrides and theme variables.
- App UI: a topic input + Start button; a live list of debates via SSE; per-debate expansion showing each round, the conclusion, key arguments, and the eval score.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. Two controls:

- **G1 — guardrail · before-agent-response**: a guardrail on the `Synthesizer` checks the conclusion for balance (represents both sides) and safety before it is persisted and shown to the user.
- **E1 — eval-event · on-decision-eval**: `SynthesisEvalConsumer` subscribes to `SynthesisRecorded` and records a quality score (balance, faithfulness to the recorded rounds) back onto the entity via `SynthesisEvaluated`. Non-blocking; informational in the UI.

## 9. Agent prompts

- `Advocate` → `prompts/advocate.md` — argue the strongest case in favour of the topic, one round at a time.
- `Critic` → `prompts/critic.md` — rebut the Advocate and argue the strongest case against.
- `Synthesizer` → `prompts/synthesizer.md` — weigh both sides and produce a balanced conclusion with key arguments.

## 10. Acceptance

Inline summary; full journeys in `reference/user-journeys.md`.

1. **J1 — Submit and watch a debate run.** `POST /api/debates {topic}` returns a `debateId`; the UI streams the debate through rounds to `CONCLUDED` with a non-empty conclusion.
2. **J2 — Inspect rounds and synthesis.** The UI shows each recorded round (advocate + critic) and the final key-arguments list.
3. **J3 — Synthesis guardrail.** A synthesis that fails the balance/safety check is blocked from persistence; the debate records a failure reason rather than showing an unbalanced conclusion.
4. **J4 — Background simulator.** Without UI interaction, `DebateSimulator` seeds a topic and a full debate completes.

---

## 11. Implementation directives

```
Create a sample named debate demonstrating the debate-multi-perspective × general
cell. Runs out of the box (no external services). Maven group io.akka.samples.
Maven artifact debate. Java package io.akka.samples.debate. Akka 3.6.0. HTTP port 9680.

Components to wire (exactly):
- 3 AutonomousAgents: Advocate (returns typed Argument{position,text}), Critic
  (returns typed Argument{position,text}), Synthesizer (returns typed
  Synthesis{conclusion, keyArguments:List<String>}). Each declares definition()
  with capability(TaskAcceptance.of(task).maxIterationsPerTask(3)). Synthesizer's
  definition wires a before-agent-response guardrail that rejects a synthesis
  failing a balance/safety check (must reference both sides; no unsafe content).
- 1 Workflow DebateModerator with steps: startStep -> roundStep -> synthesisStep.
  startStep records DebateStarted on DebateEntity. roundStep calls
  forAutonomousAgent(Advocate.class,...).runSingleTask(...) then
  forTask(taskId).result(...), then the same for Critic with the advocate's
  argument as context, records RoundRecorded, increments currentRound; if
  currentRound < maxRounds (5) and sides have not converged, self-transitions to
  roundStep again, else transitions to synthesisStep. synthesisStep calls
  Synthesizer, records SynthesisRecorded, ends. Override settings() with
  stepTimeout(60s) on roundStep and synthesisStep and
  defaultStepRecovery(maxRetries(2).failoverTo(DebateModerator::error)); the
  error step records DebateFailed. WorkflowSettings is nested in Workflow — no
  import.
- 1 EventSourcedEntity DebateEntity holding a Debate record with id, topic
  (Optional<String>), DebateStatus enum, currentRound (int), maxRounds (int),
  rounds (List<Round>), and Optional lifecycle fields conclusion, keyArguments,
  qualityScore, createdAt, concludedAt, failedReason. Events: DebateStarted,
  RoundRecorded, SynthesisRecorded, SynthesisEvaluated, DebateFailed. Commands:
  start, recordRound, recordSynthesis, recordEvaluation, fail, getDebate.
  emptyState() returns Debate.initial("") with placeholder values and NO
  commandContext() reference.
- 1 EventSourcedEntity InboundRequestQueue with a single command
  enqueueRequest(topic) emitting InboundRequestQueued.
- 1 View DebatesView with row type Debate, table updater consuming DebateEntity
  events. ONE query: getAllDebates SELECT * AS debates FROM debates_view. No
  WHERE status filter (Akka cannot auto-index enum columns) — filter
  client-side in callers.
- 1 Consumer DebateRequestConsumer subscribed to InboundRequestQueue events; on
  each event starts a DebateModerator workflow with a fresh UUID.
- 1 Consumer SynthesisEvalConsumer subscribed to DebateEntity events; on
  SynthesisRecorded, computes a quality score (balance + faithfulness to the
  recorded rounds) and calls DebateEntity.recordEvaluation. Non-blocking eval.
- 1 TimedAction DebateSimulator (every 30s, reads the next line from
  src/main/resources/sample-events/debate-topics.jsonl and calls
  InboundRequestQueue.enqueueRequest).
- 2 HttpEndpoints: DebateEndpoint at /api with debates POST, debates list
  (filter client-side from getAllDebates), single debate, SSE stream, and three
  /api/metadata/* endpoints serving the YAML/MD files from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html and
  /app/* -> static-resources/*.

Companion files:
- DebateTasks.java declaring three Task<R> constants: ADVOCATE (resultConformsTo
  Argument), CRITIC (Argument), SYNTHESIZE (Synthesis).
- Argument(String position, String text), Synthesis(String conclusion,
  List<String> keyArguments), Round(int number, String advocateArgument,
  String criticArgument).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9680
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/debate-topics.jsonl with 8 canned topic lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 2 controls (G1, E1) and matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions,
  data.types, capability.*, model.*, subjects.children; marking jurisdictions,
  declared_frameworks, organization fields, incidents, and data.residency as
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root (already authored). NO governance-mechanisms
  section. NO configuration section.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN imports for
  markdown and YAML allowed. Five tabs: Overview (eyebrow + headline, no
  subtitle, four cards Try it / How it works / Components / API contract);
  Architecture (mermaid diagrams with Lesson 24 overrides); Risk Survey (fetches
  risk-survey.yaml, renders as matrix-card with matrix-row pairs; mutes deployer
  placeholders); Eval Matrix (fetches eval-matrix.yaml, renders as matrix-card,
  one matrix-row per control, colored mechanism pill in the label column); App UI
  (submit topic, SSE list of debates, expand to show rounds + conclusion + key
  arguments + eval score). Match the governance.html visual style
  (dark/yellow/Instrument Sans/dot-grid).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (Advocate -> Argument, Critic -> Argument, Synthesizer -> Synthesis;
  see src/main/resources/mock-responses/{advocate,critic,synthesizer}.json with
  4-6 entries each). Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed to
  the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured reference
  if it does not resolve at runtime; never echoes any captured key.

Mock LLM provider — per-agent mock-response shapes:
- advocate.json: 4-6 Argument entries, position="for", text = 2-3 sentence
  argument in favour.
- critic.json: 4-6 Argument entries, position="against", text = 2-3 sentence
  rebuttal.
- synthesizer.json: 4-6 Synthesis entries, conclusion = balanced 3-4 sentence
  paragraph, keyArguments = 3-5 short bullet strings drawn from both sides.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause
  matches verbatim.
- Lesson 4: every workflow step calling an agent overrides stepTimeout(60s).
- Lesson 6: Optional<T> for every nullable field on the Debate view row record.
- Lesson 7: AutonomousAgent requires a companion DebateTasks.java.
- Lesson 8: verify model names current before locking application.conf.
- Lesson 9: run command is "/akka:build", never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9680.
- Lesson 11: never render source.platform in any user-facing surface.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is "Runs out of the box"; never T1/T2/T3/T4.
- Lesson 23: no competitor brand names anywhere user-facing.
- Lesson 24: index.html includes the mermaid state-label CSS overrides + theme
  variables (state-diagram label colour, edge-label foreignObject overflow
  visible, transitionLabelColor #cccccc).
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching by data-tab / data-panel attribute, never NodeList
  index; delete removed panels, never display:none zombies.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and 6.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
