# Sample Specification — `stock-analyst-regulated`

The natural-language input for `/akka:specify`. Running `/akka:specify @SPEC.md` from inside this folder produces the Akka project. Sections 1–11 are the spec; Section 12 chains the rest of the pipeline.

---

## 1. System name + pitch

**System name:** Stock Analysis Team.
**One-line pitch:** The user enters a stock ticker; a supervisor delegates news search, SEC 10-Q/10-K review, and summarization to worker agents, then a recommendation agent emits a disclosed BUY/HOLD/SELL recommendation that is sector-scrubbed and held for live compliance review.

## 2. What this blueprint demonstrates

The coordination pattern is delegation with a supervisor and workers: one orchestrator fans work out to specialized agents and aggregates their typed results into one recommendation. The governance pattern wires four controls onto a regulated financial-advice output — a before-agent-response guardrail that enforces disclaimer presence and a safety check on the recommendation, a sector sanitizer that scrubs fair-disclosure-sensitive phrasing from research text, a periodic drift-and-fairness watch over the recommendation distribution, and a human-over-the-loop live compliance review that can clear or retract an issued recommendation without blocking the pipeline.

## 3. User-facing flows

1. The user submits a ticker (e.g. `ACME`) in the App UI or via `POST /api/analyses`. The response carries the `analysisId`. The analysis appears in `QUEUED`, then `RESEARCHING`.
2. The supervisor delegates to the news and filings agents in parallel, then to the summary agent. The UI shows the analysis advancing through `RESEARCHING` → `SUMMARIZING` → `SANITIZING` → `RECOMMENDING`.
3. The recommendation agent emits a `BUY`/`HOLD`/`SELL` call with rationale and a regulatory disclaimer. The before-agent-response guardrail rejects any recommendation missing the disclaimer or failing the safety check. The analysis reaches `ISSUED_PENDING_REVIEW`.
4. A compliance reviewer reads the issued recommendation in the App UI and clicks Clear or Retract. The analysis moves to `CLEARED` or `RETRACTED`.
5. Without any interaction, the simulator drips canned tickers and the drift monitor periodically flags a skewed recommendation distribution on the affected analyses.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| StockAnalysisEndpoint | HttpEndpoint | Submit/list/get analyses, review actions, SSE, metadata | UI, clients | AnalysisEntity, AnalysisWorkflow, AnalysisView |
| AppEndpoint | HttpEndpoint | Serve `/` and `/app/*` static UI | Browser | static-resources |
| AnalysisWorkflow | Workflow | Supervisor: delegate research, summarize, sanitize, recommend, await review | RequestConsumer, StockAnalysisEndpoint | worker agents, AnalysisEntity |
| NewsResearchAgent | Agent | Search simulated news for the ticker | AnalysisWorkflow | AnalysisWorkflow |
| FilingsResearchAgent | Agent | Extract figures from simulated SEC 10-Q/10-K text | AnalysisWorkflow | AnalysisWorkflow |
| SummaryAgent | Agent | Summarize news + filings into one brief | AnalysisWorkflow | AnalysisWorkflow |
| RecommendationAgent | Agent | Emit BUY/HOLD/SELL with rationale + disclaimer | AnalysisWorkflow | AnalysisWorkflow |
| AnalysisEntity | EventSourcedEntity | Hold one analysis lifecycle | AnalysisWorkflow, StockAnalysisEndpoint | AnalysisView |
| InboundRequestQueue | EventSourcedEntity | Queue canned ticker requests | RequestSimulator | RequestConsumer |
| AnalysisView | View | CQRS read model of analyses | AnalysisEntity | StockAnalysisEndpoint |
| RequestConsumer | Consumer | Start one workflow per queued request | InboundRequestQueue | AnalysisWorkflow |
| RequestSimulator | TimedAction | Drip canned tickers every 30s | sample-events JSONL | InboundRequestQueue |
| DriftMonitor | TimedAction | Sample recommendation distribution, flag drift every 60s | AnalysisView | AnalysisEntity |

Names are used verbatim by `/akka:specify`.

## 5. Data model

Authoritative. See `reference/data-model.md`. The view row type is the `Analysis` record; every field that is null before its transition is declared `Optional<T>` (Lesson 6).

`Analysis` record: `id` (String), `ticker` (String), `status` (AnalysisStatus enum), and Optional lifecycle fields: `newsFindings`, `filingFindings`, `summary`, `recommendation` (BUY/HOLD/SELL), `rationale`, `disclaimer`, `sanitizedNotes`, `driftFlag` (Boolean), `reviewedBy`, `reviewNote`, plus `queuedAt`, `researchedAt`, `summarizedAt`, `sanitizedAt`, `issuedAt`, `reviewedAt`, `failedAt`.

`AnalysisStatus` enum: `QUEUED`, `RESEARCHING`, `SUMMARIZING`, `SANITIZING`, `RECOMMENDING`, `ISSUED_PENDING_REVIEW`, `CLEARED`, `RETRACTED`, `FAILED`.

Events: `AnalysisQueued`, `ResearchCompleted`, `SummaryCompleted`, `Sanitized`, `RecommendationIssued`, `ReviewCleared`, `ReviewRetracted`, `DriftFlagged`, `AnalysisFailed`.

## 6. API contract

Inline surface below; payload schemas in `reference/api-contract.md`.

```
POST /api/analyses                       -> { analysisId }
POST /api/analyses/{id}/clear            -> 200 | 404
POST /api/analyses/{id}/retract          -> 200 | 404
GET  /api/analyses ?status=...           -> { analyses: [Analysis, ...] }
GET  /api/analyses/{id}                  -> Analysis
GET  /api/analyses/sse                   -> Server-Sent Events of Analysis

GET  /api/metadata/eval-matrix           -> text/yaml
GET  /api/metadata/risk-survey           -> text/yaml
GET  /api/metadata/readme                -> text/markdown

GET  /                                    -> 302 /app/index.html
GET  /app/{*path}                         -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` — no `ui/` folder, no npm build. Browser title `<title>Akka Sample: Stock Analysis Team</title>`. Five tabs: Overview, Architecture, Risk Survey, Eval Matrix, App UI. See `reference/ui-mockup.md`.

- Tab switching matches by `data-tab` / `data-panel` attribute, never by NodeList index; removed tabs are deleted from the DOM, never `display:none`-hidden (Lesson 26).
- The Architecture tab renders the four mermaid diagrams from `PLAN.md` with the Akka theme variables AND the CSS overrides for state-diagram labels and edge-label `foreignObject` overflow (Lesson 24).
- The App UI tab: submit a ticker; live SSE list of analyses; per-analysis Clear/Retract buttons when status is `ISSUED_PENDING_REVIEW`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. Mechanisms the generated system wires:

- **G1 guardrail · before-agent-response** — on `RecommendationAgent`: reject any recommendation missing the regulatory disclaimer or failing the safety check.
- **S1 sanitizer · sector** — the workflow's sanitize step scrubs fair-disclosure-sensitive phrasing (material non-public-information patterns, selective-disclosure language) from research and summary text before the recommendation is produced.
- **E1 eval-periodic · drift-fairness-watch** — `DriftMonitor` samples recent recommendations every 60s and flags a skewed BUY/HOLD/SELL distribution.
- **H1 hotl · live-compliance-review** — issued recommendations enter `ISSUED_PENDING_REVIEW`; a compliance reviewer can Clear or Retract live. Non-blocking — the recommendation is already issued and observable.

## 9. Agent prompts

- `prompts/news-research-agent.md` — searches simulated news for the ticker and returns dated headlines with sentiment.
- `prompts/filings-research-agent.md` — extracts revenue, margin, and risk-factor figures from simulated 10-Q/10-K text.
- `prompts/summary-agent.md` — fuses news and filings into one neutral brief.
- `prompts/recommendation-agent.md` — emits a BUY/HOLD/SELL call with rationale and the mandatory regulatory disclaimer.

## 10. Acceptance

Inline below; full journeys in `reference/user-journeys.md`.

1. **Submit and recommend.** POST a ticker; within ~60s the analysis reaches `ISSUED_PENDING_REVIEW` with a non-empty recommendation, rationale, and disclaimer.
2. **Disclaimer guardrail.** A recommendation generated without the disclaimer is rejected by the before-agent-response guardrail and never persists.
3. **Live compliance review.** Clicking Retract on an issued analysis moves it to `RETRACTED` with the reviewer recorded; Clear moves it to `CLEARED`.
4. **Drift watch.** When recent recommendations skew (e.g. all BUY), the drift monitor sets `driftFlag` on the affected analyses and the UI shows the flag.

---

## 11. Implementation directives

The whole SPEC.md — Sections 1–11 — is the input to `/akka:specify @SPEC.md`. Section 11 carries the Akka-specific detail.

```
Create a sample named stock-analyst-regulated demonstrating the
delegation-supervisor-workers x finance-analysis cell. Runs out of the box
(no external services; news and SEC filings are modeled in-process). Maven
group io.akka.samples. Maven artifact stock-analyst-regulated. Java package
io.akka.samples.stockanalystregulated. Akka 3.6.0. HTTP port 9956.

Components to wire (exactly):
- 4 Agents (request/response): NewsResearchAgent.research(String ticker)
  returning a typed NewsFindings; FilingsResearchAgent.extract(String ticker)
  returning FilingFindings; SummaryAgent.summarize(ResearchBundle) returning a
  String brief; RecommendationAgent.recommend(ResearchBundle) returning a typed
  Recommendation{call, rationale, disclaimer}. Each uses
  effects().systemMessage(prompt).userMessage(input).responseAs(R.class)
  .thenReply(). Do NOT downgrade or upgrade the primitive (Lesson 1).
- 1 Workflow AnalysisWorkflow with steps: researchStep (calls NewsResearchAgent
  and FilingsResearchAgent, records ResearchCompleted), summarizeStep (calls
  SummaryAgent, records SummaryCompleted), sanitizeStep (runs sector sanitizer
  over news+filings+summary text, records Sanitized), recommendStep (calls
  RecommendationAgent; the before-agent-response guardrail enforces disclaimer
  presence + safety check; records RecommendationIssued, status
  ISSUED_PENDING_REVIEW). Override settings() with stepTimeout(60s) on
  researchStep, summarizeStep, recommendStep and
  defaultStepRecovery(maxRetries(2).failoverTo(AnalysisWorkflow::error)).
  WorkflowSettings is nested in Workflow — no import (Lesson 5).
- 1 EventSourcedEntity AnalysisEntity holding an Analysis record with id,
  ticker, AnalysisStatus enum, and Optional lifecycle fields (newsFindings,
  filingFindings, summary, recommendation, rationale, disclaimer,
  sanitizedNotes, driftFlag, reviewedBy, reviewNote, queuedAt, researchedAt,
  summarizedAt, sanitizedAt, issuedAt, reviewedAt, failedAt). Events:
  AnalysisQueued, ResearchCompleted, SummaryCompleted, Sanitized,
  RecommendationIssued, ReviewCleared, ReviewRetracted, DriftFlagged,
  AnalysisFailed. Commands: recordQueued, recordResearch, recordSummary,
  recordSanitized, recordRecommendation, clear, retract, flagDrift, fail,
  getAnalysis. emptyState() returns Analysis.initial("", "") with NO
  commandContext() reference (Lesson 3).
- 1 EventSourcedEntity InboundRequestQueue with one command
  enqueueRequest(ticker) emitting InboundRequestQueued.
- 1 View AnalysisView, row type Analysis, table updater consuming
  AnalysisEntity events. ONE query getAllAnalyses SELECT * AS analyses FROM
  analyses_view. No WHERE status filter (Akka cannot auto-index enum columns,
  Lesson 2) — filter client-side in callers.
- 1 Consumer RequestConsumer subscribed to InboundRequestQueue events; on each
  event starts an AnalysisWorkflow with a fresh UUID.
- 2 TimedActions: RequestSimulator (every 30s, reads next line from
  src/main/resources/sample-events/analysis-requests.jsonl, calls
  InboundRequestQueue.enqueueRequest); DriftMonitor (every 60s, queries
  AnalysisView.getAllAnalyses, filters ISSUED_PENDING_REVIEW/CLEARED, computes
  BUY/HOLD/SELL distribution, calls AnalysisEntity.flagDrift on affected
  analyses when one call dominates beyond a threshold).
- 2 HttpEndpoints: StockAnalysisEndpoint at /api with analyses POST, clear,
  retract, list (filter client-side from getAllAnalyses), single, SSE stream,
  and three /api/metadata/* endpoints serving YAML/MD from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html
  and /app/* -> static-resources/*.

Companion files:
- Records: NewsFindings(List<String> headlines, String sentiment),
  FilingFindings(String revenue, String margin, List<String> riskFactors),
  ResearchBundle(String ticker, NewsFindings news, FilingFindings filings,
  String summary), Recommendation(String call, String rationale, String
  disclaimer).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9956 and agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
  Verify model names are current before locking (Lesson 8).
- src/main/resources/sample-events/analysis-requests.jsonl with 8 canned
  ticker lines (e.g. {"ticker":"ACME"}).
- src/main/resources/sample-docs/<ticker>-news.md and <ticker>-10q.md for the
  in-process news and filings sources the worker agents read.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with controls G1, S1, E1, H1 and a
  matching simplified_view list. Finance-anchored: leave regulation_anchors
  empty unless a concrete instrument is cited.
- risk-survey.yaml at the project root, pre-filling sector (financial
  services / investment research), decisions, data.types, capability.*,
  model.*, subjects.children=false; marking jurisdictions, declared
  frameworks, organization fields, incidents, residency, retention as
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root per the per-blueprint structure. NO
  governance-mechanisms section, NO configuration section (Lesson 20).
- src/main/resources/static-resources/index.html — one self-contained file
  (Lesson 17). Five tabs (Overview, Architecture, Risk Survey, Eval Matrix,
  App UI). Architecture tab renders the four PLAN.md mermaid diagrams with the
  Lesson 24 theme variables AND CSS overrides. Risk Survey and Eval Matrix
  render in matrix-card/matrix-row style; mechanism pills colored per Lesson
  18. App UI: submit ticker, SSE list, Clear/Retract on
  ISSUED_PENDING_REVIEW. Match the governance.html dark/yellow/Instrument
  Sans/dot-grid style. Tab switching by attribute, no zombie panels (Lesson
  26).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding, /akka:specify inspects the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (NewsResearchAgent -> NewsFindings; FilingsResearchAgent ->
  FilingFindings; SummaryAgent -> brief String; RecommendationAgent ->
  Recommendation with a disclaimer always present so the guardrail passes; see
  src/main/resources/mock-responses/{news-research,filings-research,summary,
  recommendation}.json with 4-6 entries each). Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in
  .akka-build.yaml; /akka:build sources it before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives only in Claude session memory;
  passed to the JVM via the MCP tool's environment parameter; gone at session
  end.
- NEVER write the key value to any file Akka creates. Record only the
  reference (env-var name, file path, secrets URI), never the value.
- Bootstrap.java fails fast with a clear message naming the configured
  reference if it does not resolve at runtime; never echoes any captured key.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Lesson 1: AutonomousAgent/Agent primitive matches the spec; never silently
  swapped.
- Lesson 4: every workflow step calling an agent has an explicit stepTimeout
  (60s) — the 5s default times out LLM calls.
- Lesson 6: Optional<T> for every nullable field on the Analysis view row.
- Lesson 7: if any agent is changed to AutonomousAgent, add the companion
  Tasks.java — these workers are request/response Agents, no Tasks.java.
- Lesson 8: verify model names current before locking application.conf.
- Lesson 9: run command is "/akka:build", never "mvn akka:run".
- Lesson 10: explicit dev-mode http-port = 9956.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column, no horizontal scroll.
- Lesson 13: integration label is "Runs out of the box", never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: index.html includes mermaid state-label CSS overrides + edge
  foreignObject overflow:visible + transitionLabelColor #cccccc.
- Lesson 25: five-option key sourcing; never write the key to disk.
- Lesson 26: tab switching by data-tab/data-panel attribute; no zombie panels.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and the PLAN.md.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars or the mock provider), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
