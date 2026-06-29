# SPEC — consulting

The natural-language brief `/akka:specify` reads to generate this system. The whole file (Sections 1–12) is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Consulting Coordinator.
**One-line pitch:** The user submits a client engagement brief; the coordinator decides whether to delegate routine research to a junior researcher or hand the engagement off to a senior consultant, and the chosen worker returns a deliverable that appears in the UI.

## 2. What this blueprint demonstrates

The delegation-supervisor-workers coordination pattern with both branches in one supervisor: a `ConsultingCoordinator` agent classifies each engagement and either delegates to a junior worker or hands off to a senior worker. The governance pattern combines three mechanisms — an output guardrail on client-facing deliverables, a non-blocking compliance review of senior recommendations, and an automated quality eval on the routing decision itself.

## 3. User-facing flows

1. The user submits an engagement brief in the App UI (or a brief arrives from the intake simulator). The system creates an `Engagement` in `RECEIVED` state.
2. The `EngagementWorkflow` calls the `ConsultingCoordinator`, which returns a routing decision (`DELEGATE` or `HANDOFF`) with a complexity score and rationale. The engagement moves to `ROUTED`, and a routing eval fires.
3. On `DELEGATE`, the `JuniorResearcher` produces a research brief; the engagement moves to `RESEARCHING` then `DELIVERED`.
4. On `HANDOFF`, the `SeniorConsultant` produces a client recommendation; the engagement moves to `CONSULTING` then `DELIVERED`. A compliance reviewer can then post a non-blocking review, moving it to `COMPLIANCE_REVIEWED` (or `FLAGGED`).
5. The user watches each transition live in the App UI, including the routing rationale, eval score, deliverable, and any compliance verdict.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ConsultingCoordinator` | Agent | Classifies an engagement, returns a `RoutingDecision` | `EngagementWorkflow` | `EngagementWorkflow` |
| `JuniorResearcher` | AutonomousAgent | Produces a `ResearchBrief` for routine engagements | `EngagementWorkflow` | `EngagementWorkflow` |
| `SeniorConsultant` | AutonomousAgent | Produces a `ConsultingRecommendation` for high-stakes engagements | `EngagementWorkflow` | `EngagementWorkflow` |
| `EngagementWorkflow` | Workflow | Routes, runs the chosen worker, records delivery | `EngagementRequestConsumer`, `EngagementEndpoint` | `EngagementEntity`, agents |
| `EngagementEntity` | EventSourcedEntity | Holds engagement lifecycle state | `EngagementWorkflow`, endpoints, monitors | `EngagementsView` |
| `InboundRequestQueue` | EventSourcedEntity | Records inbound engagement briefs | `RequestSimulator`, `EngagementEndpoint` | `EngagementRequestConsumer` |
| `EngagementsView` | View | CQRS read model over `EngagementEntity` events | `EngagementEntity` | `EngagementEndpoint`, `RoutingEvalConsumer` |
| `EngagementRequestConsumer` | Consumer | Starts one workflow per inbound brief | `InboundRequestQueue` | `EngagementWorkflow` |
| `RoutingEvalConsumer` | Consumer | Scores the routing decision on `EngagementRouted` | `EngagementEntity` | `EngagementEntity` |
| `RequestSimulator` | TimedAction | Drips canned briefs every 30s | `sample-events/engagements.jsonl` | `InboundRequestQueue` |
| `EngagementEndpoint` | HttpEndpoint | REST + SSE + metadata surface | UI / clients | `InboundRequestQueue`, `EngagementEntity`, `EngagementsView` |
| `AppEndpoint` | HttpEndpoint | Serves the static UI | browser | `static-resources/` |

## 5. Data model

See `reference/data-model.md` for the full table. The `Engagement` record is the entity state and the view row type; every lifecycle field is `Optional<T>` (Lesson 6).

```
Engagement(
  String id,
  Optional<String> brief,
  EngagementStatus status,
  Optional<String> route,                 // DELEGATE | HANDOFF
  Optional<Double> complexityScore,
  Optional<String> routingRationale,
  Optional<Instant> routedAt,
  Optional<String> assignedTo,            // junior | senior
  Optional<String> deliverableTitle,
  Optional<String> deliverableContent,
  Optional<Instant> deliveredAt,
  Optional<Double> evalScore,
  Optional<String> evalNotes,
  Optional<Instant> complianceReviewedAt,
  Optional<String> complianceReviewer,
  Optional<String> complianceVerdict,     // PASS | FLAG
  Optional<String> complianceNotes
)

enum EngagementStatus { RECEIVED, ROUTED, RESEARCHING, CONSULTING, DELIVERED, COMPLIANCE_REVIEWED, FLAGGED }

events: EngagementReceived, EngagementRouted, ResearchDelivered,
        RecommendationDelivered, RoutingEvaluated, ComplianceReviewed
```

## 6. API contract

See `reference/api-contract.md` for payload schemas and the SSE event format. Top-level surface:

```
POST /api/engagements                     -> { engagementId }
POST /api/engagements/{id}/compliance     -> 200 | 404   (compliance review of a senior recommendation)
GET  /api/engagements                     -> { engagements: [Engagement, ...] }
GET  /api/engagements/{id}                -> Engagement
GET  /api/engagements/sse                 -> Server-Sent Events of Engagement

GET  /api/metadata/eval-matrix            -> text/yaml
GET  /api/metadata/risk-survey            -> text/yaml
GET  /api/metadata/readme                 -> text/markdown

GET  /                                     -> 302 /app/index.html
GET  /app/{*path}                          -> static-resources/{*path}
```

## 7. UI

Five tabs — Overview / Architecture / Risk Survey / Eval Matrix / App UI — in a single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Browser title: `<title>Akka Sample: Consulting Coordinator</title>`. See `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline (sample type), then the four cards Try it / How it works / Components / API contract. No subtitle.
- **Architecture** — the four mermaid diagrams from `PLAN.md` with the Lesson 24 mermaid CSS overrides (state-label colour, edge-label `overflow:visible`, `transitionLabelColor` `#cccccc`).
- **Risk Survey** — renders `/api/metadata/risk-survey` in `matrix-card` / `matrix-row` style; `TO_BE_COMPLETED_BY_DEPLOYER` values render muted.
- **Eval Matrix** — renders `/api/metadata/eval-matrix` in `matrix-card` / `matrix-row` style with a colored mechanism pill per control.
- **App UI** — submit an engagement brief; live SSE list of engagements; each card shows route, complexity, eval score, deliverable, and a compliance-review form on `DELIVERED` senior recommendations.

Tab switching matches by `data-tab` / `data-panel` attribute, never by NodeList index; removed tabs are deleted from the DOM, not hidden (Lesson 26).

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. Mechanisms the generated system wires:

- **G1 — output guardrail (guardrail · before-agent-response):** a guardrail on `JuniorResearcher` and `SeniorConsultant` checks the client-facing deliverable for length, disclaimer presence, and absence of unsupported guarantees before it is persisted.
- **H1 — compliance review (hotl · live-compliance-review):** after a senior recommendation is delivered, a compliance reviewer posts a non-blocking review through `POST /api/engagements/{id}/compliance`; delivery is not blocked on it.
- **E1 — routing eval (eval-event · on-decision-eval):** `RoutingEvalConsumer` fires on `EngagementRouted` to score whether the delegate-vs-handoff decision matched the brief's stakes, recording an eval score shown in the UI.

## 9. Agent prompts

- `prompts/consulting-coordinator.md` — classify the engagement and return a routing decision.
- `prompts/junior-researcher.md` — produce a concise research brief for routine engagements.
- `prompts/senior-consultant.md` — produce a client-facing recommendation for high-stakes engagements.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

- **J1** — A routine brief is delegated; a `ResearchBrief` is delivered and shown in the UI (`DELIVERED`, `assignedTo = junior`).
- **J2** — A high-stakes brief is handed off; a `ConsultingRecommendation` is delivered (`DELIVERED`, `assignedTo = senior`).
- **J3** — A compliance reviewer posts a review on a delivered senior recommendation; the engagement moves to `COMPLIANCE_REVIEWED` or `FLAGGED`.
- **J4** — Every routed engagement shows a non-null `evalScore` from the routing eval.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows. SPEC.md as a whole — Sections 1–12 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named consulting demonstrating the delegation-supervisor-workers ×
research-intel cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact consulting. Java package io.akka.samples.consulting.
Akka 3.6.0. HTTP port 9895.

Components to wire (exactly):
- 1 Agent ConsultingCoordinator: a request/response Agent with a route(String brief)
  method using effects().systemMessage(...).userMessage(...).responseAs(RoutingDecision.class)
  .thenReply(). Returns RoutingDecision{route, complexityScore, rationale}. Do NOT make this
  an AutonomousAgent — it is a single classification call.
- 2 AutonomousAgents: JuniorResearcher (returns a typed ResearchBrief{title,content}) and
  SeniorConsultant (returns a typed ConsultingRecommendation{title,content}). Each declares
  definition() with .instructions(...) and capability(TaskAcceptance.of(task).maxIterationsPerTask(3)).
  Each is wired with a before-agent-response guardrail (G1) checking the deliverable.
- 1 Workflow EngagementWorkflow with steps routeStep -> branchStep -> (delegateStep | handoffStep)
  -> deliverStep. routeStep calls componentClient.forAgent().inSession(id).method(
  ConsultingCoordinator::route).invoke(brief), writes EngagementRouted. branchStep reads the
  route: DELEGATE -> delegateStep calls forAutonomousAgent(JuniorResearcher.class,...).runSingleTask
  then forTask(taskId).result(...); HANDOFF -> handoffStep calls SeniorConsultant the same way.
  deliverStep writes ResearchDelivered or RecommendationDelivered to EngagementEntity. Override
  settings() with stepTimeout(60s) on routeStep, delegateStep, handoffStep, and
  defaultStepRecovery(maxRetries(2).failoverTo(error)).
- 1 EventSourcedEntity EngagementEntity holding the Engagement record (id, brief Optional<String>,
  EngagementStatus enum, and the Optional lifecycle fields from Section 5). Events: EngagementReceived,
  EngagementRouted, ResearchDelivered, RecommendationDelivered, RoutingEvaluated, ComplianceReviewed.
  Commands: recordReceived, recordRouting, recordResearch, recordRecommendation, recordEval,
  recordCompliance, getEngagement. emptyState() returns Engagement.initial("") with no
  commandContext() reference.
- 1 EventSourcedEntity InboundRequestQueue with command enqueueRequest(brief) emitting
  InboundRequestQueued.
- 1 View EngagementsView with row type Engagement, table updater consuming EngagementEntity events.
  ONE query: getAllEngagements SELECT * AS engagements FROM engagements_view. No WHERE status
  filter (Akka cannot auto-index enum columns) — filter client-side in callers.
- 1 Consumer EngagementRequestConsumer subscribed to InboundRequestQueue events; on each event
  starts an EngagementWorkflow with a fresh UUID.
- 1 Consumer RoutingEvalConsumer subscribed to EngagementEntity events; on EngagementRouted it
  computes an eval score (does the chosen route match the complexity score?) and calls
  EngagementEntity.recordEval. Non-blocking (E1).
- 1 TimedAction RequestSimulator (every 30s, reads next line from
  src/main/resources/sample-events/engagements.jsonl and calls InboundRequestQueue.enqueueRequest).
- 2 HttpEndpoints: EngagementEndpoint at /api with engagements (POST create, GET list filtered
  client-side, GET single, SSE stream), POST {id}/compliance, and three /api/metadata/* endpoints
  serving the YAML/MD files from src/main/resources/metadata/. AppEndpoint serving / -> 302
  /app/index.html and /app/* -> static-resources/*.

Companion files:
- ConsultingTasks.java declaring two Task<R> constants: RESEARCH (resultConformsTo ResearchBrief)
  and RECOMMEND (resultConformsTo ConsultingRecommendation).
- RoutingDecision(String route, double complexityScore, String rationale),
  ResearchBrief(String title, String content),
  ConsultingRecommendation(String title, String content),
  ComplianceReview(String reviewer, String verdict, String notes).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9895 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai (gpt-4o),
  googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/engagements.jsonl with 8 canned engagement briefs (mix of
  routine and high-stakes: market sizing, competitor scan, regulatory M&A review, etc.).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with controls G1, H1, E1 and a matching simplified_view.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data.types, capability.*,
  model.*, subjects.children; marking jurisdictions, declared_frameworks, organization fields,
  incidents, and data.residency as TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root (already authored): elevator pitch, component inventory, matrix
  cell, integration descriptor, how to run, the tabs, an ASCII architecture diagram, project
  layout, API contract, license. NO governance-mechanisms section. NO configuration section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Inline CSS + JS. Runtime CDN imports for markdown and YAML libs are
  acceptable. Five tabs: Overview, Architecture (mermaid with Lesson 24 CSS), Risk Survey,
  Eval Matrix, App UI (submit brief, SSE list, compliance-review form on delivered senior
  recommendations). Match the governance.html visual style (dark/yellow/Instrument Sans/dot-grid).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for ANTHROPIC_API_KEY /
  OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If one is set, default application.conf's
  model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random outputs
  (ConsultingCoordinator -> RoutingDecision, JuniorResearcher -> ResearchBrief, SeniorConsultant
  -> ConsultingRecommendation; see src/main/resources/mock-responses/{consulting-coordinator,
  junior-researcher,senior-consultant}.json with 4-6 entries each). Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml; /akka:build
  sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed to the JVM via
  the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE — env-var
  name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured reference if it does not
  resolve at runtime; never echoes any captured key.

Mock LLM provider — per-agent mock-response shapes when the user picks option (a):
- consulting-coordinator.json: 4-6 RoutingDecision objects, mixing route DELEGATE (low
  complexityScore ~0.2-0.4) and HANDOFF (high complexityScore ~0.7-0.9), each with a short rationale.
- junior-researcher.json: 4-6 ResearchBrief objects with a title and a 3-5 paragraph content body.
- senior-consultant.json: 4-6 ConsultingRecommendation objects with a title and a 4-6 paragraph
  content body including a standard disclaimer line.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent (JuniorResearcher, SeniorConsultant) never silently downgraded to
  Agent; ConsultingCoordinator stays a request/response Agent — match the spec verbatim.
- Lesson 4: workflow step timeouts set explicitly (60s) on every agent-calling step.
- Lesson 6: Optional<T> for every nullable lifecycle field on the Engagement view-row record.
- Lesson 7: each AutonomousAgent has its Task<R> constant in ConsultingTasks.java.
- Lesson 8: verify model names current before locking application.conf.
- Lesson 9: the run command is "/akka:build", never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9895 in application.conf.
- Lesson 11: never render source.platform anywhere.
- Lesson 12: UI fits the content column with no horizontal scroll.
- Lesson 13: integration label is "Runs out of the box" — never T1/T2/T3/T4 or "deferred".
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides and theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible, transitionLabelColor
  #cccccc).
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index; no
  hidden zombie panels — delete removed tabs.
- emptyState() never calls commandContext(). WorkflowSettings is nested inside Workflow — no import.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and `PLAN.md`.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars and the mock option), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
