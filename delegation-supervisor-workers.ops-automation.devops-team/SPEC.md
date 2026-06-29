# SPEC — devops-multi-agent

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** DevOps Multi-Agent.
**One-line pitch:** Submit a change request; a supervisor delegates infrastructure checks, deployment validation, and observability review to three specialist agents in parallel, then consolidates a change-readiness report with human approval for production targets.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work to three AutonomousAgents in parallel, gathers their assessments, and asks a fourth AutonomousAgent to consolidate a change-readiness report. The blueprint demonstrates three governance controls: a **before-tool-call guardrail** that intercepts destructive operations before any agent executes them, a **human-in-the-loop** approval gate that parks production change plans until an operator approves them, and an **operator halt** that stops any in-flight workflow on demand.

## 3. User-facing flows

The user opens the App UI tab and submits a change request (target environment, change type, description).

1. The system creates a `ChangeRequest` record in `PLANNING` and starts an `OpsWorkflow`.
2. The Coordinator parses the request and emits a structured `WorkPlan` with three subtask queries: one each for infra, deploy, and observability.
3. The workflow forks: all three specialist agents run concurrently. Each returns a typed assessment.
4. The Coordinator merges the three assessments into a `ReadinessReport { summary, infraAssessment, deployAssessment, obsAssessment, riskLevel, guardrailVerdict }`.
5. A before-tool-call guardrail checks every tool call each specialist agent proposes before execution. Any destructive call that fails the policy causes the workflow to transition to `BLOCKED`.
6. If the target environment is `production`, the consolidated report is parked; the change enters `AWAITING_APPROVAL`. An operator calls the approval API to advance it to `APPROVED` or `REJECTED`.
7. Non-production changes skip the approval gate and go directly to `APPROVED`.
8. If any specialist times out after 60 seconds, the workflow short-circuits: the Coordinator consolidates from whichever assessments returned and the change enters `DEGRADED`.
9. If an operator issues a halt signal, any in-flight `OpsWorkflow` transitions to `HALTED`.

A `PipelineSimulator` (TimedAction) drips a sample change request every 90 seconds so the App UI is non-empty on first load.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `OpsCoordinator` | `AutonomousAgent` | Parses a change request into a work plan, merges three assessments into a readiness report. | `OpsWorkflow` | returns typed result to workflow |
| `InfraAgent` | `AutonomousAgent` | Checks infrastructure drift, quota availability, and dependency health. | `OpsWorkflow` | — |
| `DeployAgent` | `AutonomousAgent` | Evaluates deployment manifests, rollback posture, and canary gate status. | `OpsWorkflow` | — |
| `ObservabilityAgent` | `AutonomousAgent` | Reviews alert thresholds, SLO burn rates, and on-call coverage. | `OpsWorkflow` | — |
| `OpsWorkflow` | `Workflow` | Coordinates parallel fan-out, guardrail enforcement, report consolidation, and approval gate. | `OpsEndpoint`, `PipelineConsumer` | `ChangeRequestEntity` |
| `ChangeRequestEntity` | `EventSourcedEntity` | Holds the change-request lifecycle (planning → in-progress → awaiting-approval / approved / rejected / degraded / blocked / halted). | `OpsWorkflow` | `OpsView` |
| `PipelineQueue` | `EventSourcedEntity` | Logs each submitted change request for replay and audit. | `OpsEndpoint`, `PipelineSimulator` | `PipelineConsumer` |
| `OpsView` | `View` | List-of-changes read model. | `ChangeRequestEntity` events | `OpsEndpoint` |
| `PipelineConsumer` | `Consumer` | Listens to `PipelineQueue` events and starts one `OpsWorkflow` per submission. | `PipelineQueue` events | `OpsWorkflow` |
| `PipelineSimulator` | `TimedAction` | Drips a sample change request every 90 s. | scheduler | `PipelineQueue` |
| `HaltMonitor` | `TimedAction` | Checks for active halt signals every 30 s; forwards halt commands to in-flight workflows. | scheduler | `OpsWorkflow` |
| `OpsEndpoint` | `HttpEndpoint` | `/api/ops/*` — submit, get, list, approve, reject, halt, SSE. | — | `OpsView`, `PipelineQueue`, `ChangeRequestEntity`, `OpsWorkflow` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record ChangeSubmission(String targetEnvironment, String changeType, String description, String requestedBy) {}

record InfraAssessment(String driftSummary, List<String> quotaWarnings, List<String> dependencyIssues,
                       RiskLevel riskLevel, Instant assessedAt) {}

record DeployAssessment(String manifestSummary, boolean rollbackAvailable, String canaryStatus,
                        RiskLevel riskLevel, Instant assessedAt) {}

record ObsAssessment(List<String> firingAlerts, double sloBurnRate, boolean onCallCovered,
                     RiskLevel riskLevel, Instant assessedAt) {}

record WorkPlan(String infraQuery, String deployQuery, String obsQuery, String targetEnvironment) {}

record ReadinessReport(String summary, InfraAssessment infraAssessment, DeployAssessment deployAssessment,
                       ObsAssessment obsAssessment, RiskLevel riskLevel,
                       String guardrailVerdict, Instant consolidatedAt) {}

record ChangeRequest(
    String changeId,
    String targetEnvironment,
    String changeType,
    String description,
    String requestedBy,
    ChangeStatus status,
    Optional<WorkPlan> workPlan,
    Optional<InfraAssessment> infraAssessment,
    Optional<DeployAssessment> deployAssessment,
    Optional<ObsAssessment> obsAssessment,
    Optional<ReadinessReport> report,
    Optional<String> failureReason,
    Optional<String> approvedBy,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ChangeStatus { PLANNING, IN_PROGRESS, AWAITING_APPROVAL, APPROVED, REJECTED, DEGRADED, BLOCKED, HALTED }

enum RiskLevel { LOW, MEDIUM, HIGH, CRITICAL }
```

### Events (on `ChangeRequestEntity`)

`ChangeCreated`, `WorkPlanEmitted`, `InfraAssessed`, `DeployAssessed`, `ObsAssessed`, `ReportConsolidated`, `ApprovalRequested`, `ChangeApproved`, `ChangeRejected`, `ChangeDegraded`, `ChangeBlocked`, `ChangeHalted`.

### Events (on `PipelineQueue`)

`ChangeSubmitted { changeId, targetEnvironment, changeType, description, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/ops/changes` — body `{ targetEnvironment, changeType, description }` → `{ changeId }`. Starts a workflow.
- `GET /api/ops/changes` — list all change requests. Optional `?status=PLANNING|IN_PROGRESS|AWAITING_APPROVAL|APPROVED|REJECTED|DEGRADED|BLOCKED|HALTED`.
- `GET /api/ops/changes/{id}` — one change request.
- `POST /api/ops/changes/{id}/approve` — body `{ approvedBy }` → advances `AWAITING_APPROVAL` to `APPROVED`.
- `POST /api/ops/changes/{id}/reject` — body `{ reason }` → advances `AWAITING_APPROVAL` to `REJECTED`.
- `POST /api/ops/halt` — body `{ reason }` → issues a system-wide halt signal; in-flight workflows enter `HALTED`.
- `GET /api/ops/changes/sse` — server-sent events stream of every change-request update.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "DevOps Multi-Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows. Three controls: G1, H1, S1.
- **App UI** — form to submit a change request (environment selector, change type, description), live list of requests with status pills, expand-row to see infra/deploy/obs assessments, readiness report, and approval controls for `AWAITING_APPROVAL` rows.

Browser title: `<title>Akka Sample: DevOps Multi-Agent</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `InfraAgent`, `DeployAgent`, `ObservabilityAgent`): intercepts every proposed tool call before execution. Calls that match a destructive-action blocklist (e.g., `deleteNamespace`, `forceDeletePod`, `disableAlert`) are rejected. Blocking. Failure → `BLOCKED`.
- **H1 — human-in-the-loop approval** (`application` flavor): when the target environment is `production`, `OpsWorkflow` parks after report consolidation; the change enters `AWAITING_APPROVAL`. An operator must call `POST /api/ops/changes/{id}/approve` or `reject` before the change proceeds.
- **S1 — operator halt** (`operator-regulator-stop`): `POST /api/ops/halt` writes a halt signal to a `HaltSignalEntity`. `HaltMonitor` (TimedAction, every 30 s) detects active signals and sends a halt command to each in-flight `OpsWorkflow`. Affected workflows enter `HALTED` and stop all pending steps.

## 9. Agent prompts

- `OpsCoordinator` → `prompts/ops-coordinator.md`. Parses a change submission into a structured work plan; later merges three assessments into a consolidated readiness report.
- `InfraAgent` → `prompts/infra-agent.md`. Evaluates infrastructure drift, quotas, and dependencies; returns `InfraAssessment`.
- `DeployAgent` → `prompts/deploy-agent.md`. Evaluates deployment manifests, rollback posture, and canary gates; returns `DeployAssessment`.
- `ObservabilityAgent` → `prompts/observability-agent.md`. Reviews alerts, SLO burn rates, and on-call coverage; returns `ObsAssessment`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a non-production change; request progresses `PLANNING → IN_PROGRESS → APPROVED` within 90 s; UI reflects each transition via SSE.
2. **J2** — Submit a production change; request parks at `AWAITING_APPROVAL`; operator approves via API; request advances to `APPROVED`.
3. **J3** — Submit a change where a specialist proposes a destructive tool call; guardrail blocks it; change enters `BLOCKED`.
4. **J4** — Issue a halt signal during an active workflow; the workflow enters `HALTED`; subsequent pipeline events are not processed.
5. **J5** — Inject a specialist timeout; change enters `DEGRADED` with partial assessments in the report.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named devops-multi-agent demonstrating the
delegation-supervisor-workers × ops-automation cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-ops-automation-devops-team.
Java package io.akka.samples.devopsmultiagent. Akka 3.6.0. HTTP port 9794.

Components to wire (exactly):
- 4 AutonomousAgents:
  * OpsCoordinator — definition() with capability(TaskAcceptance.of(PLAN).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(CONSOLIDATE).maxIterationsPerTask(3)). System prompt loaded
    from prompts/ops-coordinator.md. Returns WorkPlan{infraQuery, deployQuery, obsQuery,
    targetEnvironment} for PLAN and ReadinessReport{summary, infraAssessment, deployAssessment,
    obsAssessment, riskLevel, guardrailVerdict, consolidatedAt} for CONSOLIDATE.
  * InfraAgent — capability(TaskAcceptance.of(ASSESS_INFRA).maxIterationsPerTask(3)). System
    prompt from prompts/infra-agent.md. Returns InfraAssessment{driftSummary,
    quotaWarnings: List<String>, dependencyIssues: List<String>, riskLevel, assessedAt}.
    Before every tool call, the before-tool-call guardrail checks the proposed tool name against
    a blocklist; on match, reject and transition to ChangeBlocked.
  * DeployAgent — capability(TaskAcceptance.of(ASSESS_DEPLOY).maxIterationsPerTask(3)). System
    prompt from prompts/deploy-agent.md. Returns DeployAssessment{manifestSummary,
    rollbackAvailable: boolean, canaryStatus: String, riskLevel, assessedAt}.
    Same before-tool-call guardrail as InfraAgent.
  * ObservabilityAgent — capability(TaskAcceptance.of(ASSESS_OBS).maxIterationsPerTask(3)).
    System prompt from prompts/observability-agent.md. Returns ObsAssessment{firingAlerts:
    List<String>, sloBurnRate: double, onCallCovered: boolean, riskLevel, assessedAt}.
    Same before-tool-call guardrail as InfraAgent.

- 1 Workflow OpsWorkflow with steps:
  planStep -> [parallel] infraStep, deployStep, obsStep -> joinStep -> consolidateStep ->
  guardStep -> approvalGateStep -> emitStep.
  planStep calls forAutonomousAgent(OpsCoordinator.class, PLAN).
  infraStep, deployStep, obsStep run in parallel (CompletionStage zip); each governed by
  WorkflowSettings.builder().stepTimeout(OpsWorkflow::infraStep, ofSeconds(60))
  and matching timeouts for deployStep and obsStep. On any timeout, transition to
  degradeStep that calls consolidateStep with whichever assessments returned, then ends
  with ChangeDegraded.
  consolidateStep calls forAutonomousAgent(OpsCoordinator.class, CONSOLIDATE) with the merged
  inputs; give consolidateStep a 90s stepTimeout.
  guardStep runs the before-tool-call blocklist check over the final report; on violation
  failure end with ChangeBlocked.
  approvalGateStep: if targetEnvironment == "production", emit ApprovalRequested and pause
  the workflow waiting for an external resume signal (POST /api/ops/changes/{id}/approve or
  reject). On resume with approval, end with ChangeApproved; on rejection, end with
  ChangeRejected. Non-production environments skip to emitStep directly.
  WorkflowSettings is nested inside Workflow — no import.

- 1 EventSourcedEntity ChangeRequestEntity holding state ChangeRequest{changeId,
  targetEnvironment, changeType, description, requestedBy, ChangeStatus,
  Optional<WorkPlan> workPlan, Optional<InfraAssessment> infraAssessment,
  Optional<DeployAssessment> deployAssessment, Optional<ObsAssessment> obsAssessment,
  Optional<ReadinessReport> report, Optional<String> failureReason,
  Optional<String> approvedBy, Instant createdAt, Optional<Instant> finishedAt}.
  ChangeStatus enum: PLANNING, IN_PROGRESS, AWAITING_APPROVAL, APPROVED, REJECTED,
  DEGRADED, BLOCKED, HALTED.
  RiskLevel enum: LOW, MEDIUM, HIGH, CRITICAL.
  Events: ChangeCreated, WorkPlanEmitted, InfraAssessed, DeployAssessed, ObsAssessed,
  ReportConsolidated, ApprovalRequested, ChangeApproved, ChangeRejected, ChangeDegraded,
  ChangeBlocked, ChangeHalted.
  Commands: createChange, emitWorkPlan, attachInfra, attachDeploy, attachObs, consolidate,
  requestApproval, approve, reject, degrade, block, halt, getChange.
  emptyState() returns ChangeRequest.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity PipelineQueue with command submitChange(targetEnvironment, changeType,
  description, requestedBy) emitting ChangeSubmitted{changeId, targetEnvironment, changeType,
  description, requestedBy, submittedAt}.

- 1 EventSourcedEntity HaltSignalEntity with command issueHalt(reason) emitting
  HaltIssued{signalId, reason, issuedAt} and command clearHalt(signalId) emitting
  HaltCleared{signalId}. State holds a list of active signal ids.

- 1 View OpsView with row type ChangeRequestRow (mirrors ChangeRequest minus heavy nested
  payloads; every nullable field is Optional<T>). Table updater consumes
  ChangeRequestEntity events. ONE query getAllChanges SELECT * AS changes FROM ops_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer PipelineConsumer subscribed to PipelineQueue events; on ChangeSubmitted
  starts an OpsWorkflow with the changeId as the workflow id.

- 2 TimedActions:
  * PipelineSimulator — every 90s, reads next line from
    src/main/resources/sample-events/change-requests.jsonl and calls
    PipelineQueue.submitChange.
  * HaltMonitor — every 30s, queries HaltSignalEntity for active signals; for each active
    signal, sends halt commands to all in-flight OpsWorkflow instances in IN_PROGRESS or
    AWAITING_APPROVAL state, transitioning them to HALTED via ChangeRequestEntity.halt.

- 2 HttpEndpoints:
  * OpsEndpoint at /api with POST /ops/changes, GET /ops/changes, GET /ops/changes/{id},
    POST /ops/changes/{id}/approve, POST /ops/changes/{id}/reject, POST /ops/halt,
    GET /ops/changes/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- OpsTasks.java declaring four Task<R> constants: PLAN (WorkPlan), ASSESS_INFRA
  (InfraAssessment), ASSESS_DEPLOY (DeployAssessment), ASSESS_OBS (ObsAssessment),
  CONSOLIDATE (ReadinessReport).
- Domain records WorkPlan, InfraAssessment, DeployAssessment, ObsAssessment,
  ReadinessReport, ChangeSubmission.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9794 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/change-requests.jsonl with 8 canned change-request lines
  spanning environments (staging, dev) and change types (deploy, scale, config-update,
  rollback).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (G1 before-tool-call guardrail,
  H1 hitl application approval, S1 halt operator-regulator-stop) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function =
  ops-automation, decisions.authority_level = recommend-only (non-prod) or gated (prod),
  data.data_classes.pii = false, capabilities.* = false except content_generation = true;
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/ops-coordinator.md, prompts/infra-agent.md, prompts/deploy-agent.md,
  prompts/observability-agent.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: DevOps Multi-Agent", one-line pitch,
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills, approve/reject buttons for AWAITING_APPROVAL rows). Browser title exactly:
  <title>Akka Sample: DevOps Multi-Agent</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material. Akka
  records only the REFERENCE (env-var name, file path, secrets URI); the
  value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (ops-coordinator.json,
  infra-agent.json, deploy-agent.json, observability-agent.json), picks one
  entry pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    ops-coordinator.json — list of either WorkPlan or ReadinessReport objects.
      4–6 WorkPlan entries (infraQuery + deployQuery + obsQuery triples) and
      4–6 ReadinessReport entries (each with a 60–120 word summary,
      representative nested assessments, riskLevel = MEDIUM, guardrailVerdict = "ok").
    infra-agent.json — 4–6 InfraAssessment entries, each with a driftSummary,
      1–3 quotaWarnings, 0–2 dependencyIssues, riskLevel LOW or MEDIUM,
      assessedAt timestamp.
    deploy-agent.json — 4–6 DeployAssessment entries, each with a
      manifestSummary, rollbackAvailable = true, canaryStatus one of
      "GREEN" / "YELLOW" / "INACTIVE", riskLevel LOW or MEDIUM.
    observability-agent.json — 4–6 ObsAssessment entries, firingAlerts
      0–2 items, sloBurnRate 0.01–0.15, onCallCovered = true, riskLevel LOW.
- A MockModelProvider.seedFor(changeId) helper makes the selection
  deterministic per change id so the same request produces the same output
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s workers,
  90s consolidation); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion OpsTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9794 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible, transitionLabelColor
  #cccccc). Without these, state names render black-on-black and arrow labels clip.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index;
  delete removed panels from the DOM, do not display:none them.
- Parallel workflow steps use CompletionStage zip, NOT sequential calls.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK in
  narrative, T1/T2/T3/T4, deferred, marketing tone.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing options from Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
