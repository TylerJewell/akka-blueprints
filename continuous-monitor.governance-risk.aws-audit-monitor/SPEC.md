# SPEC — aws-audit-monitor

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** AWS Audit Monitor.
**One-line pitch:** A background worker continuously scans an AWS environment for misconfigurations, enriches each finding with AI-driven risk analysis, and holds every report for risk-officer review before publication.

## 2. What this blueprint demonstrates

The **continuous-monitor** coordination pattern wired with two governance mechanisms layered on top of two AI primitives (`FindingAnalystAgent` and `ReportCompilerAgent`). Specifically:

- A **deployer-runtime HITL** gate requiring a risk officer to explicitly publish or dismiss every compiled audit report — the AI never releases findings without human sign-off.
- A **periodic eval** sampler running every 4 hours that scores a sample of published findings against ground-truth control mappings, producing a continuous accuracy signal for the AI's analysis quality.

The result is a system where every published compliance finding has passed two independent checks: an automated AI analysis and an explicit human-review gate, with ongoing accuracy monitoring surfacing drift over time.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live finding feed: every detected AWS misconfiguration, its normalized payload, its AI-assigned severity and control mapping, and (if compiled) the draft audit report entry.
2. `AuditScanPoller` (TimedAction) ticks every 60 s and inserts new simulated findings into `FindingQueue`. (A `RequestSimulator` style — drips canned AWS findings covering IAM, S3, Security Groups, CloudTrail, and encryption gaps.)
3. For each new finding: `FindingNormalizer` (Consumer) normalizes the raw finding payload, then `FindingAnalystAgent` classifies severity, maps to control framework, and drafts a remediation recommendation.
4. Analyzed findings are accumulated by `ReportCompilerAgent` into a draft `AuditReport`. The report lands in `PENDING_REVIEW`.
5. The risk officer clicks Publish in the UI — the report transitions to PUBLISHED (simulated; no real outbound notify).
6. The risk officer clicks Dismiss with a reason — the finding or report transitions to DISMISSED.
7. `AccuracyEvalRunner` (TimedAction) ticks every 4 hours, picks N published findings without `evalScore`, calls an `AccuracyJudgeAgent` with a rubric, and writes a score back via an `EvalScored` event.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AuditScanPoller` | `TimedAction` | Drips simulated AWS findings into `FindingQueue` every 60 s. | scheduler | `FindingQueue` |
| `FindingQueue` | `EventSourcedEntity` | Append-only log of `FindingDetected` events. | `AuditScanPoller`, `AuditEndpoint` | `FindingNormalizer` |
| `FindingNormalizer` | `Consumer` | Reads `FindingDetected` events, normalizes raw AWS finding payload into canonical `NormalizedFinding`, emits `FindingNormalized` via `AuditFindingEntity`. | `FindingQueue` events | `AuditFindingEntity` |
| `FindingAnalystAgent` | `Agent` (typed, not autonomous) | Assigns severity (`CRITICAL`/`HIGH`/`MEDIUM`/`LOW`/`INFO`), maps finding to a control framework reference, and produces a one-paragraph remediation recommendation. | invoked by Workflow | returns `AnalysisResult` |
| `ReportCompilerAgent` | `AutonomousAgent` | Groups per-finding analyses into a structured audit report with an executive summary and a prioritized finding list. | invoked by Workflow | returns `CompiledReport` |
| `AccuracyJudgeAgent` | `Agent` (typed, not autonomous) | Scores the accuracy of a published finding's analysis against the rubric. | invoked by `AccuracyEvalRunner` | returns `EvalResult` |
| `FindingWorkflow` | `Workflow` | Per-finding orchestration: normalize → analyze → compile into report → wait for risk-officer review → finalize. | `FindingNormalizer` (one workflow per `FindingNormalized`) | `AuditFindingEntity`, `AuditReportEntity` |
| `AuditFindingEntity` | `EventSourcedEntity` | Lifecycle per finding: detected → normalized → analyzed → compiled → pending-review → published/dismissed. | `FindingWorkflow` | `AuditView` |
| `AuditReportEntity` | `EventSourcedEntity` | Accumulates analyzed findings into a draft report; awaits risk-officer sign-off. | `FindingWorkflow` | `AuditView` |
| `AuditView` | `View` | Read-model row per finding and per report for the UI. | `AuditFindingEntity` events, `AuditReportEntity` events | `AuditEndpoint` |
| `AccuracyEvalRunner` | `TimedAction` | Every 4 h, samples published findings without scores; calls `AccuracyJudgeAgent`; writes `EvalScored`. | scheduler | `AuditFindingEntity` |
| `AuditEndpoint` | `HttpEndpoint` | `/api/audit/*` — list findings, list reports, get finding, publish, dismiss, SSE. | — | `AuditView`, `AuditFindingEntity`, `AuditReportEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record AwsFinding(String findingId, String resourceType, String resourceArn,
                  String ruleId, String ruleName, String rawDetail, Instant detectedAt) {}

record NormalizedFinding(String resourceType, String resourceArn,
                         String ruleId, String controlDomain, String normalizedDetail,
                         List<String> affectedAttributes) {}

record AnalysisResult(Severity severity, String controlRef, String remediationSummary,
                      String confidence) {}
enum Severity { CRITICAL, HIGH, MEDIUM, LOW, INFO }

record CompiledReport(String reportId, String executiveSummary,
                      List<String> findingIds, Instant compiledAt) {}

record ReviewDecision(boolean published, String reviewedBy,
                      Optional<String> reason, Instant decidedAt) {}

record AuditFinding(
    String findingId,
    AwsFinding raw,
    Optional<NormalizedFinding> normalized,
    Optional<AnalysisResult> analysis,
    Optional<String> reportId,
    Optional<ReviewDecision> decision,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    FindingStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum FindingStatus {
    DETECTED, NORMALIZED, ANALYZED, COMPILED, PENDING_REVIEW, PUBLISHED, DISMISSED
}

record AuditReport(
    String reportId,
    Optional<CompiledReport> compiled,
    Optional<ReviewDecision> decision,
    ReportStatus status,
    Instant createdAt,
    Optional<Instant> publishedAt
) {}

enum ReportStatus { DRAFT, PENDING_REVIEW, PUBLISHED, DISMISSED }
```

Events on `AuditFindingEntity`: `FindingDetected`, `FindingNormalized`, `FindingAnalyzed`, `FindingCompiled`, `FindingPublished`, `FindingDismissed`, `EvalScored`.

Events on `AuditReportEntity`: `ReportCreated`, `ReportFindingAdded`, `ReportPublished`, `ReportDismissed`.

Events on `FindingQueue`: `FindingDetected` (re-emitted into the queue as the audit log).

See `reference/data-model.md`.

## 6. API contract

- `GET /api/audit/findings` — list all findings. Optional `?status=…&severity=…`.
- `GET /api/audit/findings/{id}` — one finding.
- `GET /api/audit/reports` — list all reports. Optional `?status=…`.
- `GET /api/audit/reports/{id}` — one report.
- `POST /api/audit/reports/{id}/publish` — body `{ reviewedBy }` → transitions PENDING_REVIEW to PUBLISHED.
- `POST /api/audit/findings/{id}/dismiss` — body `{ reviewedBy, reason }` → transitions finding to DISMISSED.
- `GET /api/audit/sse` — Server-Sent Events for every finding and report change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: AWS Audit Monitor</title>`.

App UI tab shows the **live finding feed** and a separate **reports panel**. Left column: finding cards sorted newest-first with severity chips. Right column: selected finding detail with analysis output, remediation text, and (for risk officers) Publish/Dismiss controls on the associated report.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — HITL deployer-runtime-monitoring**: no audit report is published without a risk officer clicking Publish. `FindingWorkflow` pauses every compiled report in PENDING_REVIEW; only `AuditEndpoint.publish` can advance it.
- **E1 — eval-periodic performance-monitor** (every 4 hours): scores published findings on analysis accuracy, control-mapping correctness, and remediation quality.

## 9. Agent prompts

- `FindingAnalystAgent` → `prompts/finding-analyst.md`. Typed classifier/analyst. Always returns one of the five `Severity` values plus a control reference and one-paragraph remediation.
- `ReportCompilerAgent` → `prompts/report-compiler.md`. Drafts an executive summary and prioritizes findings by severity. Never triggers outbound notifications.
- `AccuracyJudgeAgent` (used by `AccuracyEvalRunner`) → `prompts/accuracy-judge.md`. Scores a published finding's analysis on a 1–5 rubric.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Scanner drips a finding; it appears in the UI within 60 s; passes normalize → analyze → compile → PENDING_REVIEW.
2. **J2** — Risk officer clicks Publish; report transitions to PUBLISHED; no outbound network call leaves the process.
3. **J3** — Risk officer clicks Dismiss with reason; finding transitions to DISMISSED with the reason visible.
4. **J4** — AccuracyEvalRunner scores at least one PUBLISHED finding within 4 hours; the score and rationale appear in the UI.
5. **J5** — Raw AWS account identifiers (account IDs, ARNs with account segment) do not appear in the AI analysis payload — `FindingNormalizer` strips and masks them before the LLM sees the finding.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named aws-audit-monitor demonstrating the continuous-monitor × governance-risk
cell. Runs out of the box (in-memory AWS finding simulator; no real AWS integration). Maven
group io.akka.samples, artifact continuous-monitor-governance-risk-aws-audit-monitor. Java
package io.akka.samples.awsauditassistant. HTTP port 9246.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) FindingAnalystAgent — analyst/classifier. System prompt
  loaded from prompts/finding-analyst.md. Input: NormalizedFinding{resourceType, resourceArn,
  ruleId, controlDomain, normalizedDetail, affectedAttributes: List<String>}. Output:
  AnalysisResult{severity: Severity (enum CRITICAL/HIGH/MEDIUM/LOW/INFO), controlRef: String,
  remediationSummary: String, confidence: "high"|"medium"|"low"}. Defaults to HIGH severity
  under uncertainty.

- 1 AutonomousAgent ReportCompilerAgent — definition() with
  capability(TaskAcceptance.of(COMPILE_REPORT).maxIterationsPerTask(4)). System prompt from
  prompts/report-compiler.md. Input: List<AnalysisResult> + List<NormalizedFinding>.
  Output: CompiledReport{reportId, executiveSummary, findingIds: List<String>, compiledAt}.
  The agent NEVER sends notifications or emails directly.

- 1 Agent (typed, NOT autonomous) AccuracyJudgeAgent — accuracy evaluator. System prompt
  from prompts/accuracy-judge.md. Input: NormalizedFinding + AnalysisResult.
  Output: EvalResult{score: Integer 1–5, rationale: String}.

- 1 Workflow FindingWorkflow per finding with steps: normalizeStep -> analyzeStep ->
  compileStep -> awaitReviewStep -> finalizeStep. normalizeStep is inline (no LLM).
  analyzeStep wraps FindingAnalystAgent call with WorkflowSettings.builder()
  .stepTimeout(Duration.ofSeconds(20)). compileStep wraps ReportCompilerAgent with
  stepTimeout 60s. awaitReviewStep polls AuditReportEntity.getReport every 10s; on
  decision.isPresent() advances. No auto-timeout on awaitReview — reports wait indefinitely.
  finalizeStep emits FindingPublished or FindingDismissed based on decision.published.

- 3 EventSourcedEntities:
  * FindingQueue — append-only audit log of inbound findings. Command
    detect(AwsFinding) emits FindingDetected{raw}.
  * AuditFindingEntity (one per findingId) — full per-finding lifecycle. State
    AuditFinding{findingId, raw: AwsFinding{findingId, resourceType, resourceArn, ruleId,
    ruleName, rawDetail, detectedAt}, Optional<NormalizedFinding> normalized,
    Optional<AnalysisResult> analysis, Optional<String> reportId,
    Optional<ReviewDecision> decision (with published: boolean, reviewedBy,
    Optional<String> reason, decidedAt), Optional<Integer> evalScore,
    Optional<String> evalRationale, FindingStatus status, Instant createdAt,
    Optional<Instant> finishedAt}. FindingStatus enum: DETECTED, NORMALIZED, ANALYZED,
    COMPILED, PENDING_REVIEW, PUBLISHED, DISMISSED. Events: FindingDetected,
    FindingNormalized, FindingAnalyzed, FindingCompiled, FindingPublished, FindingDismissed,
    EvalScored. Commands: registerDetected, attachNormalized, attachAnalysis, attachReport,
    recordPublish, recordDismiss, recordEval, getFinding. emptyState() returns
    AuditFinding.initial("", null) without commandContext() reference.
  * AuditReportEntity (one per reportId) — accumulates findings; awaits risk-officer sign-off.
    State AuditReport{reportId, Optional<CompiledReport> compiled,
    Optional<ReviewDecision> decision, ReportStatus status, Instant createdAt,
    Optional<Instant> publishedAt}. ReportStatus enum: DRAFT, PENDING_REVIEW, PUBLISHED,
    DISMISSED. Events: ReportCreated, ReportFindingAdded, ReportPublished, ReportDismissed.
    Commands: create, addFinding, publish, dismiss, getReport.

- 1 Consumer FindingNormalizer subscribed to FindingQueue events; for each FindingDetected,
  applies a normalization pipeline: strips AWS account ID segments from ARNs, maps ruleId to
  a controlDomain, and produces NormalizedFinding with affectedAttributes. Calls
  AuditFindingEntity.registerDetected followed by attachNormalized. Then starts a
  FindingWorkflow with findingId as the workflow id.

- 1 View AuditView with two row types:
  * AuditFindingRow (mirrors AuditFinding minus raw.rawDetail for size).
  * AuditReportRow (mirrors AuditReport).
  Table updaters consume AuditFindingEntity and AuditReportEntity events. Queries:
  getAllFindings SELECT * AS findings FROM finding_view.
  getAllReports SELECT * AS reports FROM report_view.
  No WHERE filters — caller filters client-side.

- 2 TimedActions:
  * AuditScanPoller — every 60s, reads next line from
    src/main/resources/sample-events/aws-findings.jsonl and calls FindingQueue.detect.
  * AccuracyEvalRunner — every 4 hours, queries AuditView.getAllFindings, picks up to 5
    PUBLISHED findings without an evalScore (oldest-first), calls AccuracyJudgeAgent with
    a 1–5 rubric per finding, then calls AuditFindingEntity.recordEval(score, rationale)
    per finding.

- 2 HttpEndpoints:
  * AuditEndpoint at /api/audit with GET /findings, GET /findings/{id}, GET /reports,
    GET /reports/{id}, POST /reports/{id}/publish (body {reviewedBy}),
    POST /findings/{id}/dismiss (body {reviewedBy, reason}), GET /sse, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/. Publish writes ReportPublished on AuditReportEntity +
    FindingPublished on all linked AuditFindingEntities; dismiss writes FindingDismissed.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- AuditTasks.java declaring three Task<R> constants: ANALYZE (AnalysisResult),
  COMPILE_REPORT (CompiledReport), EVAL (EvalResult).
- Domain records AwsFinding, NormalizedFinding, AnalysisResult, CompiledReport,
  ReviewDecision, EvalResult.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9246 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars.
- src/main/resources/sample-events/aws-findings.jsonl with 10 canned finding lines
  covering IAM over-permissive policies, public S3 buckets, open security group rules,
  CloudTrail logging disabled, EBS encryption gaps, and root-account MFA missing.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: H1 hitl deployer-runtime-monitoring,
  E1 eval-periodic performance-monitor. Matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root with data.data_classes.pii = false,
  decisions.authority_level = advisory-only, oversight.human_in_loop = true,
  oversight.reviewer_must_approve_every_outgoing = true,
  failure.failure_modes including "missed-critical-misconfiguration" and
  "false-positive-flooding"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/finding-analyst.md, prompts/report-compiler.md, prompts/accuracy-judge.md
  loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: AWS Audit Monitor", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar with the App UI tab using a two-column layout
  (left = live finding feed with severity chips and control domain chips; right = selected
  finding detail with analysis text, remediation summary, and Publish/Dismiss controls on
  the associated report). Browser title exactly: <title>Akka Sample: AWS Audit Monitor</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the
        JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE (env-var name, file path, secrets URI); the value lives in the
  user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The message
  must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java implementing the ModelProvider interface
  with a per-agent dispatch on the agent class or Task<R> id. Each branch
  reads src/main/resources/mock-responses/<agent>.json, picks one entry
  pseudo-randomly per call, and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    finding-analyst.json — 10–12 AnalysisResult entries spanning all five
      severity levels. CRITICAL entries for public S3 + open security groups.
      HIGH for missing CloudTrail logging. MEDIUM for IAM over-permissions.
      LOW for unused credentials. INFO for tagging gaps. Each entry has
      controlRef (e.g., "CIS AWS 2.1.5"), remediationSummary (two sentences),
      and confidence. Default to HIGH under uncertainty.
    report-compiler.json — 3–4 CompiledReport entries with executiveSummary
      (two paragraphs: findings overview, recommended priority order). No
      invented remediation timelines.
    accuracy-judge.json — 6–8 EvalResult entries with score 1–5 and one-
      sentence rationales matching the rubric (severity accuracy / control
      mapping / remediation quality).
- A MockModelProvider.seedFor(findingId) helper makes per-finding selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Optional<T> for every nullable field on a View row record.
- WorkflowSettings is nested inside Workflow — no import needed.
- emptyState() never calls commandContext().
- AutonomousAgent never silently downgraded to Agent.
- FindingNormalizer runs INSIDE a Consumer before any LLM call — not inside an Agent's
  prompt and not after the LLM has seen the raw AWS finding detail.
- The agent NEVER publishes; only the risk officer Publish endpoint can transition to PUBLISHED.
- The generated static-resources/index.html must include the mermaid CSS
  overrides AND theme variables from Lesson 24 (state-diagram label colour,
  edge-label foreignObject overflow:visible, transitionLabelColor #cccccc).
  Without these, state names render black-on-black and arrow labels clip.
- Tab switching in static-resources/index.html MUST match by data-tab /
  data-panel attribute, NEVER by NodeList index (Lesson 26). No "hidden"
  zombie panels in the DOM — delete removed tabs, do not display:none them.
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var
  export block. Per Lesson 25, /akka:specify handles the key during
  generation.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK in
  narrative, T1/T2/T3/T4, marketing tone, competitor brand names.
- Lessons 1, 4, 6, 7, 8, 9, 10, 11, 12, 13, 23, 24, 25, 26 apply.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.env` file written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
