# Lead Scoring Strategy — Specification

This is the natural-language input for `/akka:specify`. Run `/akka:specify @SPEC.md` from inside this folder. Sections 1–11 together are the spec; Section 12 auto-chains the rest.

---

## 1. System name + one-line pitch

**Lead Scoring Strategy.** A lead's form responses arrive; four agents run in sequence — intake, research, scoring, strategy — and the system returns a numeric score with a tailored engagement strategy.

## 2. What this blueprint demonstrates

The coordination pattern is sequential-pipeline: four agents run one after another, each consuming the prior stage's output, with no human gate in the middle. The governance pattern wires two controls — PII sanitization of the lead's form responses before they are stored or placed in any model prompt (sanitizer, pii flavor), and an automatic eval that fires on every score (eval-event, on-decision-eval flavor) so each scoring decision carries an accuracy/fairness result.

## 3. User-facing flows

1. A lead arrives — submitted via `POST /api/leads` or dripped by `LeadSimulator`. The system records it in `NEW`, sanitizes its form PII, and starts a `LeadStrategyWorkflow`.
2. The workflow runs intake (`IntakeAgent`), turning sanitized form responses into a structured profile; the lead moves to `INTAKE`.
3. The workflow researches the company and industry (`ResearchAgent`); the lead moves to `RESEARCHED`.
4. The workflow scores the lead 0–100 (`ScoringAgent`); the lead moves to `SCORED`.
5. `ScoreEvalConsumer` reacts to the score, runs an automatic accuracy/fairness eval, and records an eval result on the lead — this does not change the lifecycle status.
6. The workflow produces a tailored engagement strategy (`StrategyAgent`); the lead moves to terminal `COMPLETE` with the strategy attached.

These become the acceptance journeys in `reference/user-journeys.md`.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `IntakeAgent` | AutonomousAgent | Turns sanitized form responses into a structured profile | `LeadStrategyWorkflow` | `LeadEntity` |
| `ResearchAgent` | AutonomousAgent | Researches the company and industry | `LeadStrategyWorkflow` | `LeadEntity` |
| `ScoringAgent` | AutonomousAgent | Produces a 0–100 score + rationale | `LeadStrategyWorkflow` | `LeadEntity` |
| `StrategyAgent` | AutonomousAgent | Drafts a tailored engagement strategy | `LeadStrategyWorkflow` | `LeadEntity` |
| `LeadStrategyWorkflow` | Workflow | Orchestrates intake → research → score → strategy | `InboundLeadConsumer` | the four agents, `LeadEntity` |
| `LeadEntity` | EventSourcedEntity | Per-lead durable lifecycle | workflow, eval consumer | `LeadsView` |
| `InboundLeadQueue` | EventSourcedEntity | Records each inbound lead submission | `LeadSimulator`, `LeadEndpoint` | `InboundLeadConsumer` |
| `LeadsView` | View | List read model, SSE source | `LeadEntity` events | `LeadEndpoint` |
| `InboundLeadConsumer` | Consumer | Starts a workflow per inbound lead | `InboundLeadQueue` events | `LeadStrategyWorkflow` |
| `ScoreEvalConsumer` | Consumer | Runs an automatic eval on each scored lead | `LeadEntity` `LeadScored` events | `LeadEntity` |
| `LeadSimulator` | TimedAction | Drips sample leads every 30s | sample-events file | `InboundLeadQueue` |
| `LeadEndpoint` | HttpEndpoint | `/api` surface + SSE + metadata | clients | entities, view |
| `AppEndpoint` | HttpEndpoint | Serves `/` redirect and `/app/*` | browser | static-resources |
| `Bootstrap` | service-setup | Schedules the TimedAction on startup | — | — |

Component count: 4 autonomous-agent · 1 workflow · 2 event-sourced-entity · 1 view · 2 consumer · 1 timed-action · 2 http-endpoint · 1 service-setup.

## 5. Data model

Full detail in `reference/data-model.md`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

```java
record Lead(
  String id,
  Optional<String> company,
  Optional<String> contactName,        // PII — stored only after sanitization
  Optional<String> formResponses,      // raw form text — stored only after sanitization
  LeadStatus status,
  Optional<Instant> intakeAt,
  Optional<String>  intakeProfile,
  Optional<Instant> researchedAt,
  Optional<String>  researchBrief,
  Optional<Instant> scoredAt,
  Optional<Integer> score,
  Optional<String>  scoreRationale,
  Optional<Instant> evalAt,
  Optional<Integer> evalScore,          // eval-event result: confidence 0-100
  Optional<String>  evalFlags,          // accuracy/fairness flags, comma-joined or "none"
  Optional<Instant> strategizedAt,
  Optional<String>  engagementStrategy
) {
  static Lead initial(String id, String company) { /* all Optional.empty() */ }
  Lead applyEvent(LeadEvent e);
}
```

`LeadStatus` enum: `NEW · INTAKE · RESEARCHED · SCORED · COMPLETE`.

`LeadEvent` sealed interface, variants: `LeadIntake`, `LeadResearched`, `LeadScored`, `ScoreEvaluated`, `StrategyProduced`.

Auxiliary records: `IntakeProfile(String company, String contactName, String profile)`, `ResearchBrief(String brief)`, `LeadScore(int score, String rationale)`, `EngagementStrategy(String strategy)`. The eval result is computed in Java by `ScoreEvalConsumer` and carried by the `ScoreEvaluated` event.

## 6. API contract

Full schemas in `reference/api-contract.md`. Surface:

```
POST /api/leads                      -> { leadId }
GET  /api/leads ?status=...          -> { leads: [Lead, ...] }
GET  /api/leads/{leadId}             -> Lead
GET  /api/leads/sse                  -> Server-Sent Events of Lead
GET  /api/metadata/eval-matrix       -> text/yaml
GET  /api/metadata/risk-survey       -> text/yaml
GET  /api/metadata/readme            -> text/markdown
GET  /                               -> 302 /app/index.html
GET  /app/{*path}                    -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (Lesson 17 — no `ui/`, no npm). Browser title `<title>Akka Sample: Lead Scoring Strategy</title>`. Five tabs: Overview, Architecture, Risk Survey, Eval Matrix, App UI. Tab switching matches by `data-tab` / `data-panel` attribute, never by NodeList index, with no hidden zombie panels (Lesson 26). Mermaid diagrams on the Architecture tab use the Akka theme variables plus the CSS overrides for state labels and edge-label `foreignObject` overflow (Lesson 24). The App UI tab submits a lead, streams the live lead list via SSE, and expands each row to show the score, the eval result, and the engagement strategy once `COMPLETE`. Full description in `reference/ui-mockup.md`.

## 8. Governance

Controls in `eval-matrix.yaml`; deployer survey in `risk-survey.yaml`.

- **S1 — sanitizer · pii.** `PiiSanitizer` redacts and normalizes the contact name and form responses inside `intakeStep` before the lead is stored and before any agent prompt is built; no raw PII reaches a model call.
- **E1 — eval-event · on-decision-eval.** `ScoreEvalConsumer` subscribes to `LeadScored` events and runs an automatic accuracy/fairness eval on each score, recording an eval result (confidence + flags) on the lead so every scoring decision carries a check.

## 9. Agent prompts

One file per agent under `prompts/`:

- `prompts/intake-agent.md` — turn sanitized form responses into a structured profile.
- `prompts/research-agent.md` — research the company and industry into a short brief.
- `prompts/scoring-agent.md` — assign a 0–100 score with a short rationale.
- `prompts/strategy-agent.md` — draft a tailored engagement strategy.

## 10. Acceptance

Full journeys in `reference/user-journeys.md`. Inlined:

1. **Submit and score.** `POST /api/leads` returns a `leadId`; within ~90s the lead reaches `SCORED` with a non-empty `score` and `scoreRationale` over SSE.
2. **Auto-eval on score.** Each `SCORED` lead gains a non-empty `evalScore` and `evalFlags` from `ScoreEvalConsumer` shortly after scoring.
3. **Strategy produced.** The lead reaches terminal `COMPLETE` with a non-empty `engagementStrategy`.
4. **Background load.** Sample leads dripped by `LeadSimulator` progress to `COMPLETE` with no manual submission.

## 11. Implementation directives

```
Create a sample named lead-scoring-strategy demonstrating the sequential-pipeline ×
sales-marketing cell. Runs out of the box (no external services). Maven group
io.akka.samples, artifact lead-scoring-strategy. Java package
io.akka.samples.leadscoringstrategy. Akka 3.6.0. HTTP port 9643.

Components to wire (exactly):
- 4 AutonomousAgents, each with definition() declaring
  capability(TaskAcceptance.of(task).maxIterationsPerTask(3)):
  - IntakeAgent    -> IntakeProfile(company, contactName, profile)
  - ResearchAgent  -> ResearchBrief(brief)
  - ScoringAgent   -> LeadScore(score, rationale)
  - StrategyAgent  -> EngagementStrategy(strategy)
- 1 Workflow LeadStrategyWorkflow with steps intakeStep -> researchStep ->
  scoreStep -> strategyStep. Linear, no human gate. Each agent step calls
  forAutonomousAgent(Agent.class, role+leadId).runSingleTask(...) then
  forTask(taskId).result(...). intakeStep runs PiiSanitizer over the raw
  contactName and formResponses BEFORE building the agent prompt and BEFORE
  LeadEntity.recordIntake. Override settings() with stepTimeout(60s) on
  intakeStep, researchStep, scoreStep, strategyStep. WorkflowSettings is nested
  in Workflow — no import.
- 1 EventSourcedEntity LeadEntity holding a Lead record with id, company
  (Optional), contactName (Optional), formResponses (Optional), LeadStatus enum,
  and the Optional lifecycle fields from Section 5. Events: LeadIntake,
  LeadResearched, LeadScored, ScoreEvaluated, StrategyProduced. Commands:
  recordIntake, recordResearch, recordScore, recordEvaluation, recordStrategy,
  getLead. ScoreEvaluated sets evalScore/evalFlags/evalAt without changing
  status. emptyState() returns Lead.initial("", "") with no commandContext()
  reference.
- 1 EventSourcedEntity InboundLeadQueue (single instance "default") with one
  command enqueueLead(rawSource) emitting InboundLeadQueued.
- 1 View LeadsView with row type Lead, table updater consuming LeadEntity
  events. ONE query getAllLeads: SELECT * AS leads FROM leads_view. No WHERE
  status filter (Akka cannot auto-index enum columns) — filter client-side.
  Provide streamAllLeads for SSE.
- 1 Consumer InboundLeadConsumer subscribed to InboundLeadQueue events; on each
  event starts a LeadStrategyWorkflow with a fresh UUID.
- 1 Consumer ScoreEvalConsumer subscribed to LeadEntity events; on LeadScored it
  computes an eval in plain Java (score in 0-100 range, rationale non-empty,
  flag rationale text that references protected attributes such as age, gender,
  race, religion, or national origin), then calls LeadEntity.recordEvaluation
  with an evalScore (0-100 confidence) and evalFlags ("none" or a comma-joined
  list). No LLM call in the consumer.
- 1 TimedAction LeadSimulator (every 30s, reads the next line from
  src/main/resources/sample-events/inbound-leads.jsonl and calls
  InboundLeadQueue.enqueueLead).
- 2 HttpEndpoints: LeadEndpoint at /api with leads submit, leads list (filter
  client-side from getAllLeads), single lead, SSE stream, and three
  /api/metadata/* endpoints serving the YAML/MD files from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html
  and /app/* -> static-resources/*.

Companion files:
- LeadStrategyTasks.java declaring four Task<R> constants: INTAKE
  (resultConformsTo IntakeProfile), RESEARCH (ResearchBrief), SCORE (LeadScore),
  STRATEGY (EngagementStrategy).
- IntakeProfile(String company, String contactName, String profile),
  ResearchBrief(String brief), LeadScore(int score, String rationale),
  EngagementStrategy(String strategy).
- PiiSanitizer.java — a static helper redacting emails/phones and normalizing
  the contact name and form responses; called from intakeStep before any agent
  prompt is built.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9643 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/inbound-leads.jsonl with 8 canned leads.
- src/main/resources/metadata/{eval-matrix.yaml, risk-survey.yaml, README.md}
  (copies of the root files for the endpoint to serve from the classpath).
- eval-matrix.yaml at the project root with the 2 controls (S1, E1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions,
  data.types (includes contact PII), capability.*, model.*, subjects.children;
  marking deployer fields TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root per the authoring guide. NO governance-
  mechanisms section, NO configuration section.
- src/main/resources/static-resources/index.html — single self-contained HTML
  file. Five tabs (Overview / Architecture / Risk Survey / Eval Matrix /
  App UI). Match the governance.html visual style (dark / yellow / Instrument
  Sans / dot-grid).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding, inspect the environment for ANTHROPIC_API_KEY /
  OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set, default
  application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (IntakeAgent -> IntakeProfile; ResearchAgent -> ResearchBrief;
  ScoringAgent -> LeadScore; StrategyAgent -> EngagementStrategy; write 4-6
  entries each to src/main/resources/mock-responses/<agent>.json). Sets
  model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed
  to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Record only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured
  reference if it does not resolve at runtime; never echoes any captured key.

Constraints — see AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: workflow step timeouts set explicitly on every agent-calling step.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: every AutonomousAgent has its Task<R> constant in LeadStrategyTasks.
- Lesson 8: verify model names current before locking them in.
- Lesson 9: run command is "/akka:build", never "mvn akka:run".
- Lesson 10: explicit dev-mode http-port (9643).
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px column with no horizontal scroll.
- Lesson 13: integration label is "Runs out of the box".
- Lesson 23: no competitor brand names anywhere user-facing.
- Lesson 24: mermaid state-label colour + edge-label overflow CSS overrides.
- Lesson 25: five-option key sourcing; never write the key to disk.
- Lesson 26: tab switching by data-tab/data-panel attribute, no zombie panels.
- emptyState() never calls commandContext().
```

## 12. Post-scaffolding workflow

When `/akka:specify` finishes scaffolding the specification into `specs/features/`, do not stop and wait for the user. Continue automatically:

1. Run `/akka:plan` — produce the architectural plan. Accept defaults; pick the most conservative option when ambiguous, consistent with Sections 4 and the PLAN.
2. Run `/akka:tasks` — break the plan into tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and summarise all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

When `/akka:build` reports the service is running, output just the listening URL (`http://localhost:9643`) and a one-line summary of any failures from step 3. Stop earlier only on a hard error you cannot work around without the user — an unresolved API key, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
