# SPEC — recruitment-high-risk

The natural-language brief `/akka:specify @SPEC.md` reads to generate this system. Sections 1–11 together are the input; Section 12 chains the rest of the workflow.

---

## 1. System name + pitch

**System name:** Recruitment Pipeline.
**One-line pitch:** A user submits a job requisition in the App UI; the system sources candidate profiles, redacts protected attributes, screens each against the requisition, and returns a match score with a written rationale — pausing on any sourcing call that targets a non-allowlisted domain and on an operator halt.

## 2. What this blueprint demonstrates

The sequential-pipeline coordination pattern: three agents run in a fixed order inside one workflow, each consuming the prior stage's typed output. The governance pattern layers four controls onto a high-risk hiring use case — a before-tool-call guardrail that blocks sourcing against domains whose terms of service forbid automated collection, a special-category sanitizer that strips protected attributes before any candidate is scored, an operator/regulator halt switch that stops new evaluations, and a deployer monitor that aggregates outcome metrics for human-on-loop review.

## 3. User-facing flows

1. The user submits a requisition (role title, requirements, candidate count) in the App UI. The system creates one candidate evaluation per sourced profile and shows them in a live list.
2. Each candidate moves through SOURCED → SANITIZED → SCREENED → MATCHED. The user watches status and the final match score and rationale stream in.
3. If a sourcing call targets a domain not on the allowlist, the candidate enters BLOCKED with the blocked domain shown; no profile data is collected.
4. The user (acting as operator or regulator) clicks Halt. New requisitions and in-flight sourcing stop; existing candidates show HALTED. Resume releases them.
5. The Monitoring panel shows the running match-score distribution and pass/reject counts the deployer is expected to watch.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `SourcingAgent` | AutonomousAgent | Sources candidate profiles via a source-candidates tool; the tool call passes a before-tool-call guardrail | `RecruitmentWorkflow` | `RecruitmentWorkflow` |
| `ScreeningAgent` | AutonomousAgent | Screens a redacted profile against the requisition requirements | `RecruitmentWorkflow` | `RecruitmentWorkflow` |
| `MatchingAgent` | AutonomousAgent | Produces a 0–100 match score and rationale | `RecruitmentWorkflow` | `RecruitmentWorkflow` |
| `RecruitmentWorkflow` | Workflow | Runs sourceStep → sanitizeStep → screenStep → matchStep | `RequisitionConsumer` | `CandidateEntity` |
| `CandidateEntity` | EventSourcedEntity | Holds one candidate's evaluation lifecycle | `RecruitmentWorkflow`, `RecruitmentEndpoint` | `CandidatesView` |
| `RequisitionQueue` | EventSourcedEntity | Inbound requisition queue | `RecruitmentEndpoint`, `RequisitionSimulator` | `RequisitionConsumer` |
| `SystemControlEntity` | KeyValueEntity | Halt switch (operator/regulator stop) | `RecruitmentEndpoint` | `RecruitmentWorkflow`, `RequisitionConsumer` |
| `CandidatesView` | View | CQRS read model over `CandidateEntity` | `CandidateEntity` | `RecruitmentEndpoint`, `DeployerMonitor` |
| `RequisitionConsumer` | Consumer | Starts one workflow per queued requisition (skips when halted) | `RequisitionQueue` | `RecruitmentWorkflow` |
| `RequisitionSimulator` | TimedAction | Drips canned requisitions every 45 s | — | `RequisitionQueue` |
| `DeployerMonitor` | TimedAction | Every 60 s aggregates outcome metrics for human-on-loop oversight | `CandidatesView` | `RecruitmentEndpoint` |
| `RecruitmentEndpoint` | HttpEndpoint | `/api` requisitions, candidates, halt/resume, monitoring, metadata | clients | `RequisitionQueue`, `CandidateEntity`, `SystemControlEntity` |
| `AppEndpoint` | HttpEndpoint | Serves the UI | browser | static resources |

## 5. Data model

See `reference/data-model.md`. Authoritative records:

- `Candidate` (entity state + view row): `String id`, `Optional<String> requisitionId`, `Optional<String> roleTitle`, `CandidateStatus status`, `Optional<Instant> sourcedAt`, `Optional<String> sourceHandle`, `Optional<Instant> sanitizedAt`, `Optional<String> redactedCategories`, `Optional<Instant> screenedAt`, `Optional<String> screenSummary`, `Optional<Boolean> screenPass`, `Optional<Instant> matchedAt`, `Optional<Double> matchScore`, `Optional<String> matchRationale`, `Optional<Instant> rejectedAt`, `Optional<String> rejectReason`, `Optional<Instant> blockedAt`, `Optional<String> blockedDomain`, `Optional<Instant> haltedAt`. Every lifecycle field is `Optional<T>` (Lesson 6).
- `CandidateStatus` enum: `SOURCED, SANITIZED, SCREENED, MATCHED, REJECTED, BLOCKED, HALTED`.
- Events: `CandidateSourced`, `ProfileSanitized`, `CandidateScreened`, `CandidateMatched`, `CandidateRejected`, `SourcingBlocked`, `EvaluationHalted`.
- `emptyState()` returns `Candidate.initial("")` with no `commandContext()` reference (Lesson 3).

## 6. API contract

See `reference/api-contract.md` for payload schemas. Top-level surface:

```
POST /api/requisitions              -> { candidateIds: [...] }
GET  /api/candidates                -> { candidates: [Candidate, ...] }
GET  /api/candidates/{id}           -> Candidate
GET  /api/candidates/sse            -> Server-Sent Events of Candidate
POST /api/system/halt               -> { halted: true }
POST /api/system/resume             -> { halted: false }
GET  /api/system/status             -> { halted }
GET  /api/monitoring                -> { evaluated, matched, rejected, blocked, avgScore }
GET  /api/metadata/eval-matrix      -> text/yaml
GET  /api/metadata/risk-survey      -> text/yaml
GET  /api/metadata/readme           -> text/markdown
GET  /                              -> 302 /app/index.html
GET  /app/{*path}                   -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm. Browser title `<title>Akka Sample: Recruitment Pipeline</title>`. Five tabs (Overview / Architecture / Risk Survey / Eval Matrix / App UI) — see `reference/ui-mockup.md`. Tab switching matches by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). The Architecture tab renders the PLAN.md mermaid diagrams with the state-label CSS overrides and theme variables from Lesson 24. The App UI tab submits a requisition, shows a live SSE candidate list, surfaces blocked/halted states, and carries the Halt/Resume control plus the monitoring summary.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. Mechanisms the generated system wires:

- **G1 guardrail · before-tool-call** — the `SourcingAgent` source-candidates tool checks the target domain against an allowlist before each call; a non-allowlisted domain (terms-of-service-protected) blocks the call and drives the candidate to BLOCKED.
- **S1 sanitizer · special-category** — `sanitizeStep` strips protected attributes (name, gender, age, ethnicity, photo, marital status) from the sourced profile before screening or scoring; the redacted categories are recorded.
- **HT1 halt · operator-regulator-stop** — `SystemControlEntity` holds a halt flag; the workflow's first step and `RequisitionConsumer` check it; `/api/system/halt` flips it.
- **HO1 hotl · deployer-runtime-monitoring** — `DeployerMonitor` aggregates outcome metrics every 60 s for human-on-loop oversight as required of a deployer of a hiring system.

## 9. Agent prompts

- `prompts/sourcing-agent.md` — sources candidate profiles from the simulated sourcing tool and returns a typed `SourcedProfile`.
- `prompts/screening-agent.md` — screens a redacted profile against the requisition and returns a pass/fail with a summary.
- `prompts/matching-agent.md` — scores the fit 0–100 and writes a short rationale.

## 10. Acceptance

See `reference/user-journeys.md`. Passing journeys:

1. **Source and match.** Submitting a requisition produces candidates that reach MATCHED with a non-null score and rationale within ~60 s.
2. **Sourcing blocked.** A requisition flagged to target a non-allowlisted domain drives the candidate to BLOCKED with the domain shown; no profile fields are populated.
3. **Special-category redaction.** Every SANITIZED candidate records redacted categories; no protected attribute reaches the screening or matching stage.
4. **Operator halt.** After Halt, new requisitions do not start workflows and in-flight candidates show HALTED; Resume releases new work.

---

## 11. Implementation directives

The whole SPEC.md (Sections 1–11) is the input to `/akka:specify @SPEC.md`. This section carries the Akka-specific detail Sections 1–10 don't repeat.

```
Create a sample named recruitment-high-risk demonstrating the
sequential-pipeline × hr-recruiting cell. Runs out of the box (no external
services; the candidate source is modeled inside the service from canned data).
Maven group io.akka.samples, artifact recruitment-high-risk. Java package
io.akka.samples.recruitmenthighrisk. Akka 3.6.0. HTTP port 9674.

Components to wire (exactly):
- 3 AutonomousAgents: SourcingAgent (returns SourcedProfile{handle, rawText});
  ScreeningAgent (returns ScreeningResult{pass, summary}); MatchingAgent
  (returns MatchResult{score, rationale}). Each declares definition() with
  capability(TaskAcceptance.of(task).maxIterationsPerTask(3)). SourcingAgent
  exposes a source-candidates tool; a before-tool-call guardrail validates the
  target domain against an allowlist (in application.conf) and blocks any
  non-allowlisted, terms-of-service-protected domain.
- 1 Workflow RecruitmentWorkflow with steps sourceStep -> sanitizeStep ->
  screenStep -> matchStep. sourceStep calls forAutonomousAgent(SourcingAgent
  .class,...).runSingleTask(...) then forTask(taskId).result(...); on a
  guardrail block, write SourcingBlocked and end. sanitizeStep runs
  ProfileSanitizer.redact(profile) (no LLM) and writes ProfileSanitized.
  screenStep and matchStep call their agents. The first step checks
  SystemControlEntity; if halted, write EvaluationHalted and end. Override
  settings() with stepTimeout(60s) on sourceStep, screenStep, matchStep and
  defaultStepRecovery(maxRetries(2).failoverTo(error)).
- 1 EventSourcedEntity CandidateEntity holding the Candidate record (id,
  Optional requisitionId, Optional roleTitle, CandidateStatus enum, and the
  Optional lifecycle fields in Section 5). Events: CandidateSourced,
  ProfileSanitized, CandidateScreened, CandidateMatched, CandidateRejected,
  SourcingBlocked, EvaluationHalted. Commands: recordSourced, recordSanitized,
  recordScreened, recordMatched, reject, block, halt, getCandidate.
  emptyState() returns Candidate.initial("") with no commandContext() call.
- 1 EventSourcedEntity RequisitionQueue with command enqueue(Requisition)
  emitting RequisitionQueued.
- 1 KeyValueEntity SystemControlEntity holding { boolean halted } with
  commands halt(), resume(), status().
- 1 View CandidatesView, row type Candidate, table updater consuming
  CandidateEntity events. ONE query: getAllCandidates SELECT * AS candidates
  FROM candidates_view. No WHERE status filter (Akka cannot auto-index enum
  columns) — filter client-side in callers.
- 1 Consumer RequisitionConsumer subscribed to RequisitionQueue events; on each
  event checks SystemControlEntity.status; if not halted, starts a
  RecruitmentWorkflow per sourced slot with a fresh UUID.
- 2 TimedActions: RequisitionSimulator (every 45s, reads next line from
  src/main/resources/sample-events/requisitions.jsonl and calls
  RequisitionQueue.enqueue); DeployerMonitor (every 60s, queries
  CandidatesView.getAllCandidates, computes evaluated/matched/rejected/blocked
  counts and average score, stores the snapshot for /api/monitoring).
- 2 HttpEndpoints: RecruitmentEndpoint at /api (requisitions, candidates list
  filtered client-side from getAllCandidates, single candidate, SSE stream,
  system halt/resume/status, monitoring, and three /api/metadata/* endpoints
  serving files from src/main/resources/metadata/). AppEndpoint serving
  / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- RecruitmentTasks.java declaring three Task<R> constants: SOURCE
  (resultConformsTo SourcedProfile), SCREEN (ScreeningResult), MATCH
  (MatchResult).
- Records: SourcedProfile(String handle, String rawText),
  ScreeningResult(boolean pass, String summary),
  MatchResult(double score, String rationale),
  Requisition(String roleTitle, String requirements, int candidateCount,
  boolean forceBlockedDomain).
- ProfileSanitizer.java — a plain helper that removes name, gender, age,
  ethnicity, photo, and marital-status signals from rawText and returns the
  redacted text plus the list of categories it removed.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9674; a sourcing.allowlist HOCON list of permitted domains; and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/requisitions.jsonl with 8 canned
  requisitions, at least one with forceBlockedDomain true.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 4 controls (G1, S1, HT1, HO1)
  and a matching simplified_view list.
- risk-survey.yaml at the project root, pre-filling sector, decisions,
  data classes, capabilities, model family, oversight; marking deployer fields
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root (already authored).
- src/main/resources/static-resources/index.html — one self-contained file,
  inline CSS + JS, runtime CDN imports for markdown/YAML/mermaid acceptable.
  Five tabs: Overview, Architecture (renders PLAN.md mermaid with Lesson 24
  CSS overrides), Risk Survey, Eval Matrix, App UI. Match the governance.html
  visual style (dark / yellow accent / Instrument Sans / dot-grid).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding, /akka:specify inspects the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent shape-correct
  outputs (SourcingAgent → SourcedProfile, ScreeningAgent → ScreeningResult,
  MatchingAgent → MatchResult; see src/main/resources/mock-responses/
  {sourcing-agent,screening-agent,matching-agent}.json with 4–6 entries each).
  Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed
  to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, or secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured
  reference if it does not resolve at runtime; never echoes any captured key.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; the extends
  clause matches the spec verbatim.
- Lesson 4: every agent-calling workflow step sets an explicit stepTimeout.
- Lesson 6: Optional<T> for every nullable lifecycle field on the view row.
- Lesson 7: each AutonomousAgent has its Task<R> constant in RecruitmentTasks.
- Lesson 8: verify model names are current before locking them.
- Lesson 9: the run command is "/akka:build", never "mvn akka:run".
- Lesson 10: explicit dev-mode http-port (9674) in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column without horizontal scroll.
- Lesson 13: descriptive integration label, never T1/T2/T3/T4 or "deferred".
- Lesson 23: no competitor brand names anywhere user-facing (no LinkedIn,
  no Selenium, no CrewAI in UI/README/SPEC/PLAN/yaml — those live only in the
  corpus-internal id and source fields).
- Lesson 24: static-resources/index.html includes the mermaid state-label CSS
  overrides, edge-label foreignObject overflow:visible, and transitionLabelColor
  #cccccc.
- Lesson 25: five-option key sourcing; never write a key value to disk.
- Lesson 26: tab switching by data-tab / data-panel attribute, never NodeList
  index; no hidden zombie panels — delete removed tabs, do not display:none them.
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
