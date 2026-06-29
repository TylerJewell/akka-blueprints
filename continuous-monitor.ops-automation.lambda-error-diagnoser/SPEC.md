# SPEC — lambda-error-diagnoser

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Lambda Error Diagnoser.
**One-line pitch:** A background worker polls Lambda error logs, diagnoses each root cause with an AI agent, and scores every diagnosis at the moment of the incident.

## 2. What this blueprint demonstrates

The **continuous-monitor** coordination pattern with one governance mechanism layered on a single AI primitive (`DiagnosisAgent`). Specifically:

- An **on-incident eval** fires inside `IncidentWorkflow` immediately after `DiagnosisAgent` returns its result — the score is part of the same workflow execution that produced the diagnosis, not an asynchronous background job.

The on-incident positioning means every operator sees both the diagnosis and its confidence score together. There is no gap between "AI said X" and "how good was X".

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live incident board: every detected error, its normalised payload, its root-cause diagnosis, and the eval score for that diagnosis.
2. `LogPoller` (TimedAction) ticks every 20 s and inserts new simulated Lambda error events into `LogQueue`. (A `RequestSimulator` style — drips canned entries.)
3. For each new error event: `LogNormalizer` (Consumer) extracts structured error context, then `IncidentWorkflow` starts.
4. Inside the workflow: `DiagnosisAgent` classifies the root cause and suggests a fix. Immediately after, `EvalJudge` scores the diagnosis (1–5). The incident lands in RESOLVED.
5. The operator can dismiss an incident as a false positive from the UI — the incident transitions to DISMISSED.
6. The operator can re-open a DISMISSED incident if triage reveals it was genuine.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `LogPoller` | `TimedAction` | Drips simulated Lambda error events into `LogQueue` every 20 s. | scheduler | `LogQueue` |
| `LogQueue` | `EventSourcedEntity` | Append-only log of `ErrorDetected` events. | `LogPoller`, `IncidentEndpoint` | `LogNormalizer` |
| `LogNormalizer` | `Consumer` | Reads `ErrorDetected` events, extracts structured error context, emits `ErrorNormalised` via `IncidentEntity`. | `LogQueue` events | `IncidentEntity` |
| `DiagnosisAgent` | `Agent` (typed, not autonomous) | Classifies the error into a root-cause category and recommends one concrete fix action. | invoked by Workflow | returns `DiagnosisResult` |
| `EvalJudge` | `Agent` (typed, not autonomous) | Scores the diagnosis for correctness and specificity on a 1–5 rubric, immediately after diagnosis. | invoked by Workflow | returns `EvalResult` |
| `IncidentWorkflow` | `Workflow` | Per-incident orchestration: diagnose → eval → finalise. | `LogNormalizer` (one workflow per `ErrorNormalised`) | `IncidentEntity` |
| `IncidentEntity` | `EventSourcedEntity` | Lifecycle per incident: detected → normalised → diagnosed → evaluated → resolved/dismissed/reopened. | `IncidentWorkflow`, `IncidentEndpoint` | `IncidentView` |
| `IncidentView` | `View` | Read-model row per incident for the UI. | `IncidentEntity` events | `IncidentEndpoint` |
| `IncidentEndpoint` | `HttpEndpoint` | `/api/incidents/*` — list, get, dismiss, reopen, SSE. | — | `IncidentView`, `IncidentEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record LambdaLogEntry(
    String logId,
    String functionName,
    String functionArn,
    String errorType,
    String errorMessage,
    String stackTraceExcerpt,
    boolean coldStart,
    String awsRegion,
    Instant detectedAt
) {}

record NormalisedError(
    String functionName,
    String errorCategory,    // "timeout" | "oom" | "unhandled-exception" | "dependency-failure" | "config-error"
    String errorMessage,
    String stackTraceExcerpt,
    boolean coldStart,
    Severity severity        // enum
) {}

enum Severity { LOW, MEDIUM, HIGH, CRITICAL }

record DiagnosisResult(
    String rootCause,
    String fixSuggestion,
    String confidence,       // "high" | "medium" | "low"
    String reasoning
) {}

record EvalResult(
    int score,               // 1–5
    String rationale
) {}

record Incident(
    String incidentId,
    LambdaLogEntry raw,
    Optional<NormalisedError> normalised,
    Optional<DiagnosisResult> diagnosis,
    Optional<EvalResult> eval,
    IncidentStatus status,
    Optional<String> dismissReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum IncidentStatus {
    DETECTED, NORMALISED, DIAGNOSED, EVALUATED, RESOLVED, DISMISSED, REOPENED
}
```

Events on `IncidentEntity`: `ErrorDetected`, `ErrorNormalised`, `DiagnosisProduced`, `DiagnosisEvaluated`, `IncidentResolved`, `IncidentDismissed`, `IncidentReopened`.

Events on `LogQueue`: `ErrorDetected` (re-emitted as the audit log).

See `reference/data-model.md`.

## 6. API contract

- `GET /api/incidents` — list all incidents. Optional `?status=…&severity=…`.
- `GET /api/incidents/{id}` — one incident.
- `POST /api/incidents/{id}/dismiss` — body `{ dismissedBy, reason }` → transitions to DISMISSED.
- `POST /api/incidents/{id}/reopen` — body `{ reopenedBy }` → transitions back to REOPENED.
- `GET /api/incidents/sse` — Server-Sent Events for every incident change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Lambda Error Diagnoser</title>`.

App UI tab is the most distinctive: it shows a **live incident board** with severity-banded rows and an inline diagnosis + eval score chip per incident. Dismiss is a single-click action with an optional reason textarea.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — on-incident eval** (`eval-event`, flavor `on-incident-reporter`): immediately after `DiagnosisAgent` returns, `IncidentWorkflow` calls `EvalJudge` with the normalised error + diagnosis. The resulting score and rationale are stored in the incident record and surfaced alongside the diagnosis. No diagnosis ever reaches an operator without a score.

## 9. Agent prompts

- `DiagnosisAgent` → `prompts/diagnosis-agent.md`. Typed classifier + fix recommender. Always returns one `DiagnosisResult`.
- `EvalJudge` → `prompts/eval-judge.md`. Scores the diagnosis on a 1–5 rubric for correctness and specificity. Never suggests an alternative fix — only evaluates.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips an error log; incident appears in the UI within 20 s; passes normalise → diagnose → eval → RESOLVED.
2. **J2** — Operator clicks Dismiss with a reason; incident transitions to DISMISSED with reason visible.
3. **J3** — Operator re-opens a DISMISSED incident; status transitions to REOPENED.
4. **J4** — Every RESOLVED incident carries a non-null eval score; no incident reaches the UI board without one.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named lambda-error-diagnoser demonstrating the continuous-monitor × ops-automation
cell. Runs out of the box (in-memory log source; no real CloudWatch integration). Maven group
io.akka.samples. Artifact id continuous-monitor-ops-automation-lambda-error-diagnoser.
Java package io.akka.samples.lambdaerroranalysisagent. HTTP port 9283.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) DiagnosisAgent — root-cause classifier + fix recommender.
  System prompt loaded from prompts/diagnosis-agent.md. Input:
  NormalisedError{functionName, errorCategory, errorMessage, stackTraceExcerpt, coldStart,
  severity: Severity}. Output: DiagnosisResult{rootCause: String, fixSuggestion: String,
  confidence: "high"|"medium"|"low", reasoning: String}.

- 1 Agent (typed, NOT autonomous) EvalJudge — diagnosis quality scorer.
  System prompt loaded from prompts/eval-judge.md. Input: NormalisedError + DiagnosisResult.
  Output: EvalResult{score: Integer 1–5, rationale: String}.

- 1 Workflow IncidentWorkflow per incident with steps:
  diagnoseStep → evalStep → finaliseStep.
  diagnoseStep wraps DiagnosisAgent with WorkflowSettings.builder()
  .stepTimeout(Duration.ofSeconds(15)). evalStep wraps EvalJudge with stepTimeout 10s.
  finaliseStep emits IncidentResolved. On step timeout in diagnoseStep, emit IncidentResolved
  with a DiagnosisResult{rootCause="diagnosis-timeout", confidence="low"} and score=1.

- 2 EventSourcedEntities:
  * LogQueue — append-only audit log of inbound errors. Command accept(LambdaLogEntry)
    emits ErrorDetected{raw}.
  * IncidentEntity (one per incidentId) — full per-incident lifecycle. State
    Incident{incidentId, raw: LambdaLogEntry, Optional<NormalisedError> normalised,
    Optional<DiagnosisResult> diagnosis, Optional<EvalResult> eval,
    IncidentStatus status, Optional<String> dismissReason, Instant createdAt,
    Optional<Instant> finishedAt}. IncidentStatus enum: DETECTED, NORMALISED, DIAGNOSED,
    EVALUATED, RESOLVED, DISMISSED, REOPENED. Events: ErrorDetected, ErrorNormalised,
    DiagnosisProduced, DiagnosisEvaluated, IncidentResolved, IncidentDismissed,
    IncidentReopened. Commands: registerRaw, attachNormalised, attachDiagnosis, attachEval,
    markResolved, dismiss, reopen, getIncident. emptyState() returns Incident.initial("")
    without commandContext() reference.

- 1 Consumer LogNormalizer subscribed to LogQueue events; for each ErrorDetected,
  extracts NormalisedError from the raw LambdaLogEntry (maps errorType to errorCategory,
  infers severity from errorCategory + coldStart, takes first 10 lines of stackTraceExcerpt),
  calls IncidentEntity.registerRaw followed by attachNormalised. Then starts an
  IncidentWorkflow with incidentId as the workflow id.

- 1 View IncidentView with row type IncidentRow (mirrors Incident minus the raw log body).
  Table updater consumes IncidentEntity events. ONE query getAllIncidents
  SELECT * AS incidents FROM incident_view ORDER BY createdAt DESC. No WHERE filter — caller
  filters client-side.

- 2 TimedActions:
  * LogPoller — every 20s, reads next line from src/main/resources/sample-events/
    lambda-errors.jsonl and calls LogQueue.accept.

- 2 HttpEndpoints:
  * IncidentEndpoint at /api with GET /incidents, GET /incidents/{id},
    POST /incidents/{id}/dismiss (body {dismissedBy, reason}),
    POST /incidents/{id}/reopen (body {reopenedBy}), GET /incidents/sse, and
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/. Dismiss writes IncidentDismissed; reopen writes
    IncidentReopened.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- IncidentTasks.java declaring two Task<R> constants: DIAGNOSE (DiagnosisResult),
  EVALUATE (EvalResult).
- Domain records LambdaLogEntry, NormalisedError, DiagnosisResult, EvalResult.
- Severity enum: LOW, MEDIUM, HIGH, CRITICAL.
- IncidentStatus enum: DETECTED, NORMALISED, DIAGNOSED, EVALUATED, RESOLVED, DISMISSED, REOPENED.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9283 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars.
- src/main/resources/sample-events/lambda-errors.jsonl with 10 canned error lines
  covering timeout, OOM, unhandled-exception, dependency-failure, and config-error categories
  with mixed severity.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 1 control: E1 eval-event on-incident-reporter.
  Matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root with sector = ops-automation pre-filled,
  decisions.authority_level = advisory-only, oversight.human_in_loop = false (the eval is
  automated; no human gate before RESOLVED), failure.failure_modes including
  "incorrect-root-cause-diagnosis" and "missed-critical-error"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/diagnosis-agent.md, prompts/eval-judge.md loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: Lambda Error Diagnoser", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar with the App UI tab using a two-column layout
  (left = live incident list sorted by severity then newest-first, with severity badge and
  eval score chip; right = selected incident detail with normalised error fields, full
  diagnosis text, fix suggestion, and Dismiss / Re-open buttons). Browser title exactly:
  <title>Akka Sample: Lambda Error Diagnoser</title>. No subtitle on the Overview tab.

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
    diagnosis-agent.json — 10–12 DiagnosisResult entries spanning all five
      errorCategory values (timeout, oom, unhandled-exception, dependency-failure,
      config-error). Each entry has a rootCause (1–2 sentences), a fixSuggestion
      (one actionable step), confidence ("high"/"medium"/"low"), and a reasoning
      sentence. Distribute confidence proportionally: 50% high, 30% medium, 20% low.
    eval-judge.json — 8–10 EvalResult entries with score 1–5 and rationale
      sentences matching the rubric (correctness / specificity / actionability).
- A MockModelProvider.seedFor(incidentId) helper makes per-incident selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Optional<T> for every nullable field on a View row record.
- WorkflowSettings is nested inside Workflow — no import needed.
- emptyState() never calls commandContext().
- EvalJudge is a typed Agent — NOT an AutonomousAgent.
- The eval step fires INSIDE IncidentWorkflow — not in a separate TimedAction.
  On-incident means synchronous with the workflow that produced the diagnosis.
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
