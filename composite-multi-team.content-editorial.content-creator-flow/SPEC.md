# SPEC — content-creator-flow

The natural-language brief `/akka:specify` reads to generate this system. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Content Creator Flow.
**One-line pitch:** A user types a single topic into the App UI; three writer agents produce a research report, a blog post, and a LinkedIn post; a brand reviewer blocks anything off-brand before it surfaces; a quality evaluator scores the finished campaign against brand guidelines.

## 2. What this blueprint demonstrates

The composite-multi-team coordination pattern: one orchestrating workflow fans a single topic out to a coordinated set of writer agents (research, blog, LinkedIn), then funnels their outputs through shared review and evaluation stages. The governance pattern pairs a before-response guardrail — a brand reviewer that blocks content failing brand-safety rules — with a decision-time evaluation event that scores each completed campaign against brand guidelines.

## 3. User-facing flows

1. The user enters a topic in the App UI and submits. A campaign appears in `RECEIVED` state with a generated id.
2. The campaign moves to `RESEARCHING`; the research agent produces a short report. The UI shows the report once present.
3. The campaign moves to `DRAFTING`; the blog and LinkedIn agents each produce their artifact from the topic and the research report.
4. The campaign moves to `REVIEWING`; the brand reviewer inspects all three outputs. If any output is off-brand, the campaign moves to `BLOCKED` with a reason and the outputs are withheld from the public view.
5. If review passes, the campaign moves to `EVALUATING`; the quality evaluator scores the campaign against brand guidelines and records notes.
6. The campaign reaches `COMPLETED`. The UI shows the three outputs, the brand verdict, and the quality score.
7. Without any user action, the topic simulator drips canned topics that each run the same pipeline.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| ResearchAgent | AutonomousAgent | Produce a short research report from a topic | ContentWorkflow | CampaignEntity |
| BlogAgent | AutonomousAgent | Produce a blog post from topic + research | ContentWorkflow | CampaignEntity |
| LinkedInAgent | AutonomousAgent | Produce a LinkedIn post from topic + research | ContentWorkflow | CampaignEntity |
| BrandReviewer | Agent | Brand-safety review of the three outputs (guardrail) | ContentWorkflow | CampaignEntity |
| QualityEvaluator | Agent | Score the completed campaign against brand guidelines (eval-event) | ContentWorkflow | CampaignEntity |
| ContentWorkflow | Workflow | Orchestrate research → draft → review → evaluate → complete | TopicConsumer, ContentEndpoint | All agents, CampaignEntity |
| CampaignEntity | EventSourcedEntity | Hold campaign state, outputs, verdict, score | ContentWorkflow | CampaignsView |
| InboundTopicQueue | EventSourcedEntity | Durable queue of inbound topics | TopicSimulator, ContentEndpoint | TopicConsumer |
| CampaignsView | View | CQRS read model of campaigns for the UI | CampaignEntity | ContentEndpoint |
| TopicConsumer | Consumer | Start a ContentWorkflow per queued topic | InboundTopicQueue | ContentWorkflow |
| TopicSimulator | TimedAction | Drip canned topics every 30s | sample-events file | InboundTopicQueue |
| ContentEndpoint | HttpEndpoint | Submit topic, list/stream campaigns, serve metadata | UI | InboundTopicQueue, CampaignsView |
| AppEndpoint | HttpEndpoint | Serve the static UI | browser | static-resources |

Names matter. `/akka:specify` uses them verbatim.

## 5. Data model

Authoritative — see `reference/data-model.md`. Summary:

`Campaign` record (CampaignEntity state and CampaignsView row): `String id`, `Optional<String> topic`, `CampaignStatus status`, `Optional<String> researchReport`, `Optional<String> blogPost`, `Optional<String> linkedInPost`, `Optional<String> brandVerdict`, `Optional<String> brandNotes`, `Optional<Double> qualityScore`, `Optional<String> qualityNotes`, plus `Optional<Instant>` lifecycle stamps `createdAt`, `researchedAt`, `draftedAt`, `reviewedAt`, `evaluatedAt`, `completedAt`, `blockedAt`, and `Optional<String> blockReason`. Every field nullable before its transition is `Optional<T>` (Lesson 6).

`CampaignStatus` enum: `RECEIVED`, `RESEARCHING`, `DRAFTING`, `REVIEWING`, `EVALUATING`, `COMPLETED`, `BLOCKED`.

Events: `TopicReceived`, `ResearchCompleted`, `ContentDrafted`, `BrandReviewPassed`, `BrandReviewBlocked`, `QualityEvaluated`, `CampaignCompleted`.

## 6. API contract

Inline surface; payload schemas in `reference/api-contract.md`.

```
POST /api/campaigns                 -> { campaignId }
GET  /api/campaigns                 -> { campaigns: [Campaign, ...] }
GET  /api/campaigns/{id}            -> Campaign
GET  /api/campaigns/sse             -> Server-Sent Events of Campaign
GET  /api/metadata/eval-matrix      -> text/yaml
GET  /api/metadata/risk-survey      -> text/yaml
GET  /api/metadata/readme           -> text/markdown
GET  /                              -> 302 /app/index.html
GET  /app/{*path}                   -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` — no `ui/` folder, no npm build (Lesson 17). Five tabs — see `reference/ui-mockup.md`. Browser title: `<title>Akka Sample: Content Creator Flow</title>`.

Tabs: Overview, Architecture, Risk Survey, Eval Matrix, App UI. Tab switching matches by `data-tab` / `data-panel` attribute, never by NodeList index, with no hidden zombie panels left in the DOM (Lesson 26). The Architecture tab renders the mermaid diagrams with the theme variables and CSS overrides from Lesson 24 (state-label colour, edge-label `foreignObject` overflow visible). The App UI tab submits a topic and streams campaigns over SSE, showing per-campaign outputs, brand verdict, and quality score.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. Two controls:

- **G1 (guardrail · before-agent-response)** — the brand reviewer inspects every writer output before it is committed to the campaign; off-brand content moves the campaign to `BLOCKED` and is withheld from the public view.
- **E1 (eval-event · on-decision-eval)** — when review passes, the quality evaluator scores the campaign against brand guidelines and the score is recorded on the `QualityEvaluated` event for the UI.

## 9. Agent prompts

- `prompts/research-agent.md` — produce a short, factual research report on the topic.
- `prompts/blog-agent.md` — write a blog post from the topic and research.
- `prompts/linkedin-agent.md` — write a concise LinkedIn post from the topic and research.
- `prompts/brand-reviewer.md` — judge whether the outputs respect brand-safety rules; return a pass/block verdict with a reason.
- `prompts/quality-evaluator.md` — score the completed campaign against brand guidelines and explain the score.

## 10. Acceptance

Inline; full detail in `reference/user-journeys.md`.

1. **Submit a topic and watch a full campaign complete.** POST a topic; within ~60s the campaign reaches `COMPLETED` with research, blog, and LinkedIn outputs, a brand verdict of pass, and a numeric quality score.
2. **Off-brand topic is blocked.** A topic the brand reviewer flags moves the campaign to `BLOCKED` with a reason; the public campaign view withholds the outputs.
3. **Background load from the simulator.** With no UI interaction, the simulator seeds topics; each runs the same pipeline to `COMPLETED`.
4. **UI renders governance.** The Eval Matrix tab shows two controls with mechanism pills; the Risk Survey tab shows pre-filled answers with deployer placeholders muted.

---

## 11. Implementation directives

The whole SPEC.md (Sections 1–11) is the input to `/akka:specify @SPEC.md`. The directives below carry the Akka-specific detail Sections 1–10 don't repeat.

```
Create a sample named content-creator-flow demonstrating the
composite-multi-team x content-editorial cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
content-creator-flow. Java package io.akka.samples.contentcreatorflow.
Akka 3.6.0. HTTP port 9293.

Components to wire (exactly):
- 3 AutonomousAgents: ResearchAgent (returns typed ResearchReport{summary}),
  BlogAgent (returns typed BlogPost{title,body}), LinkedInAgent (returns
  typed LinkedInPost{body}). Each declares definition() with
  .instructions(...) loaded from the matching prompts/*.md and
  capability(TaskAcceptance.of(task).maxIterationsPerTask(3)). Each needs a
  companion Task<R> constant (Lesson 7).
- 2 Agents (request/response): BrandReviewer with a method
  review(ReviewInput) returning typed BrandVerdict{passed,reason};
  QualityEvaluator with a method evaluate(EvalInput) returning typed
  QualityResult{score,notes}. Use effects().systemMessage(...).userMessage(...)
  .responseAs(...).thenReply().
- 1 Workflow ContentWorkflow with steps researchStep -> draftStep ->
  reviewStep -> evaluateStep -> completeStep, plus a blockedStep. researchStep
  calls forAutonomousAgent(ResearchAgent.class,...).runSingleTask(...) then
  forTask(taskId).result(...), writes ResearchCompleted. draftStep runs
  BlogAgent and LinkedInAgent (sequentially is acceptable) and writes
  ContentDrafted. reviewStep calls BrandReviewer; on passed=false transitions
  to blockedStep (writes BrandReviewBlocked) and ends; on passed=true writes
  BrandReviewPassed and continues. evaluateStep calls QualityEvaluator and
  writes QualityEvaluated. completeStep writes CampaignCompleted. Override
  settings() with stepTimeout(60s) on researchStep, draftStep, reviewStep,
  evaluateStep and a defaultStepRecovery(maxRetries(2).failoverTo(error)).
  WorkflowSettings is nested in Workflow — no import (Lesson 5).
- 1 EventSourcedEntity CampaignEntity holding a Campaign record (fields per
  Section 5; every nullable lifecycle field Optional<T>). Commands:
  recordResearch, recordDraft, recordReviewPassed, recordReviewBlocked,
  recordEvaluation, recordCompletion, getCampaign. Events listed in Section 5.
  emptyState() returns Campaign.initial("") with no commandContext() reference
  (Lesson 3).
- 1 EventSourcedEntity InboundTopicQueue with a single command
  enqueueTopic(topic) emitting TopicEnqueued.
- 1 View CampaignsView with row type Campaign, table updater consuming
  CampaignEntity events. ONE query: getAllCampaigns SELECT * AS campaigns FROM
  campaigns_view. No WHERE status filter — Akka cannot auto-index enum columns
  (Lesson 2); filter client-side in callers.
- 1 Consumer TopicConsumer subscribed to InboundTopicQueue events; on each
  TopicEnqueued starts a ContentWorkflow with a fresh UUID campaign id and
  the topic.
- 1 TimedAction TopicSimulator (every 30s, reads the next line from
  src/main/resources/sample-events/topics.jsonl and calls
  InboundTopicQueue.enqueueTopic).
- 2 HttpEndpoints: ContentEndpoint at /api with campaigns (POST create -> 302
  the topic into InboundTopicQueue then return campaignId; GET list filtered
  client-side from getAllCampaigns; GET single; SSE stream) and three
  /api/metadata/* endpoints serving YAML/MD from src/main/resources/metadata/.
  AppEndpoint serving / -> 302 /app/index.html and /app/* ->
  static-resources/*.

Companion files:
- Records: ResearchReport(String summary), BlogPost(String title, String
  body), LinkedInPost(String body), ReviewInput(String topic, String
  researchReport, String blogPost, String linkedInPost),
  BrandVerdict(boolean passed, String reason), EvalInput(String topic, String
  blogPost, String linkedInPost), QualityResult(double score, String notes).
- ContentTasks.java declaring Task<R> constants RESEARCH (ResearchReport),
  BLOG (BlogPost), LINKEDIN (LinkedInPost).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9293 and agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
  Verify model names are current before locking (Lesson 8).
- src/main/resources/sample-events/topics.jsonl with 8 canned topic lines,
  at least one clearly off-brand so the block path is exercised.
- src/main/resources/metadata/{eval-matrix.yaml,risk-survey.yaml,README.md}
  copied from the project root for the endpoint to serve from classpath.
- eval-matrix.yaml at the project root with the 2 controls (G1, E1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data
  classes, capabilities, model family and oversight; marking deployer-specific
  fields TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root (already authored): elevator pitch, component
  inventory, integration descriptor, how to run, the tabs, project layout,
  API contract, license. No governance-mechanisms section, no configuration
  section (Lesson 20).
- src/main/resources/static-resources/index.html — a single self-contained
  file (Lesson 17). Inline CSS + JS; runtime CDN imports for markdown and YAML
  are acceptable. Five tabs: Overview, Architecture, Risk Survey, Eval Matrix,
  App UI. Match the governance.html visual style (dark / yellow accent /
  Instrument Sans / dot-grid).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify inspects the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (ResearchAgent -> ResearchReport, BlogAgent -> BlogPost, LinkedInAgent
  -> LinkedInPost, BrandReviewer -> BrandVerdict, QualityEvaluator ->
  QualityResult; write src/main/resources/mock-responses/{research-agent,
  blog-agent,linkedin-agent,brand-reviewer,quality-evaluator}.json with 4-6
  entries each). Sets model-provider = mock.
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

Mock LLM provider — per-agent response shapes when the user picks option (a):
- research-agent.json: ResearchReport with a 2-4 sentence summary.
- blog-agent.json: BlogPost with a title and a 3-5 paragraph body.
- linkedin-agent.json: LinkedInPost with a 1-2 paragraph body.
- brand-reviewer.json: BrandVerdict; most entries passed=true, at least one
  passed=false with a reason, so the block path is reachable in mock mode.
- quality-evaluator.json: QualityResult with a score in [0,1] and short notes.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause
  matches verbatim.
- Lesson 4: every agent-calling workflow step sets an explicit stepTimeout.
- Lesson 6: Optional<T> for every nullable lifecycle field on the view row.
- Lesson 7: each AutonomousAgent has a companion Task<R> in ContentTasks.java.
- Lesson 8: verify model names are current before writing application.conf.
- Lesson 9: the run command is "/akka:build", never "mvn akka:run".
- Lesson 10: explicit dev-mode http-port 9293 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column, no horizontal scroll.
- Lesson 13: integration label is "Runs out of the box", never T1/T2/T3/T4.
- Lesson 23: no competitor brand names anywhere user-facing.
- Lesson 24: mermaid state-label colour + edge-label foreignObject overflow
  CSS overrides in index.html.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching by data-tab / data-panel attribute, no zombie
  panels.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and the PLAN.md in this folder.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL (`http://localhost:9293/`) and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — a missing API key (offer the three valid env vars and the mock option from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
