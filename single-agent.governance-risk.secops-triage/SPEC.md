# SPEC — secops-triage

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** SecOps Vulnerability Triage Agent.
**One-line pitch:** A security analyst submits a raw vulnerability finding; one AI agent reads the enriched finding (passed as a task attachment, never as inline prompt text) and returns a structured triage verdict — `CRITICAL_IMMEDIATE` / `HIGH_SCHEDULED` / `MEDIUM_MONITORED` / `LOW_ACCEPTED` with a risk rationale and a recommended remediation action.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the governance-risk domain. One `VulnerabilityTriageAgent` (AutonomousAgent) carries the entire triage decision; the surrounding components only enrich its input, gate its remediation proposals, and audit its outputs. Three governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** intercepts every remediation action the agent proposes before it can execute — production systems cannot be touched without explicit analyst sign-off. If the agent proposes an action the guardrail identifies as a direct remediation call (patch deployment, firewall-rule change, service restart), the guardrail blocks the call and routes it to an analyst approval queue.
- A **human-in-the-loop approval step** inside `TriageWorkflow` that gates the approved remediation: the workflow pauses at `PENDING_APPROVAL`, the analyst approves or rejects via the UI, and the workflow resumes only on an explicit `ApprovalGranted` or `ApprovalRejected` event.
- A **periodic drift evaluator** (`eval-periodic`, `drift-fairness-watch`) that samples the last N triaged findings every 6 hours, computes the severity distribution, and compares it against the calibration baseline stored at startup. A statistically significant shift in the distribution raises a `DriftAlertRaised` event visible in the UI.

The blueprint shows that a single-agent pattern does not mean "ungoverned" — three independent checks sit at different points around the one decision-making LLM call.

## 3. User-facing flows

The user opens the App UI tab.

1. The analyst pastes a raw finding (CVE id, CVSS score, affected asset, description) into the **New Finding** form (or picks one of three seeded examples — a critical container escape, a high-severity SQL injection, a medium-severity misconfiguration).
2. The analyst clicks **Ingest finding**. The UI POSTs to `/api/findings` and receives a `findingId`.
3. The card appears in the live list in `INGESTED` state. Within ~1 s, it transitions to `ENRICHED` — the enriched finding is visible in the card detail, with asset criticality, existing mitigations, and CVE metadata.
4. Within ~10–30 s, the workflow's `triageStep` completes. The card transitions to `TRIAGING` then `VERDICT_RECORDED`. The verdict appears: a priority badge (`CRITICAL_IMMEDIATE` / `HIGH_SCHEDULED` / `MEDIUM_MONITORED` / `LOW_ACCEPTED`), a 2–4 sentence risk rationale, and the recommended remediation action.
5. If the verdict is `CRITICAL_IMMEDIATE` or `HIGH_SCHEDULED`, the remediation action requires analyst approval. The card transitions to `PENDING_APPROVAL`. The analyst sees an approve/reject button pair.
6. The analyst clicks **Approve**. The workflow resumes, executes the remediation placeholder, and the card transitions to `REMEDIATED`.
7. If the analyst clicks **Reject**, the card transitions to `REMEDIATION_REJECTED` and a reason field is recorded.
8. For `MEDIUM_MONITORED` / `LOW_ACCEPTED` verdicts, no approval step is needed; the card transitions directly to `MONITORED` or `ACCEPTED` after `VERDICT_RECORDED`.
9. The periodic drift evaluator runs in the background; if a drift alert is raised, a banner appears at the top of the App UI tab.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `FindingEndpoint` | `HttpEndpoint` | `/api/findings/*` — ingest, list, get, SSE, approve/reject; serves `/api/metadata/*`. | — | `FindingEntity`, `ApprovalEntity`, `FindingView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `FindingEntity` | `EventSourcedEntity` | Per-finding lifecycle: ingested → enriched → triaging → verdict → pending-approval/remediated/rejected/monitored/accepted/failed. Source of truth. | `FindingEndpoint`, `FindingEnricher`, `TriageWorkflow` | `FindingView` |
| `ApprovalEntity` | `EventSourcedEntity` | Per-finding analyst approval state: pending → granted/rejected. | `FindingEndpoint` | `TriageWorkflow` |
| `FindingEnricher` | `Consumer` | Subscribes to `FindingIngested` events; attaches asset criticality, existing mitigations, and CVE metadata; calls `FindingEntity.attachEnriched`. | `FindingEntity` events | `FindingEntity` |
| `TriageWorkflow` | `Workflow` | One workflow per finding. Steps: `awaitEnrichedStep` → `triageStep` → `approvalGateStep` (conditional) → `remediationStep`. | started by `FindingEnricher` once enriched event lands | `VulnerabilityTriageAgent`, `FindingEntity`, `ApprovalEntity` |
| `VulnerabilityTriageAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the enriched finding as a task attachment and returns `TriageVerdict`. | invoked by `TriageWorkflow` | returns verdict |
| `RemediationGuardrail` | supporting class | before-tool-call hook wired on `VulnerabilityTriageAgent`; intercepts remediation tool calls and routes to approval queue. | invoked inside agent loop | `ApprovalEntity` |
| `DriftEvaluator` | supporting class | Periodic rule-based scorer sampling severity distribution. Runs on a timer inside `DriftCheckWorkflow`. No LLM call. | `FindingView` | `FindingEntity` (via `DriftCheckWorkflow`) |
| `DriftCheckWorkflow` | `Workflow` | Scheduled every 6 h; runs `DriftEvaluator`; emits `DriftAlertRaised` if drift detected. | timer | `FindingEntity`, `FindingView` |
| `FindingView` | `View` | Read model: one row per finding for the UI. | `FindingEntity` events | `FindingEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record RawFinding(
    String cveId,
    double cvssScore,
    String affectedAsset,
    String description,
    String submittedBy,
    Instant submittedAt
) {}

record AssetContext(
    String assetId,
    String assetCriticality,       // TIER1, TIER2, TIER3
    boolean internetFacing,
    String existingMitigations,
    String ownerTeam
) {}

record EnrichedFinding(
    RawFinding raw,
    AssetContext assetContext,
    String threatIntelSummary,     // exploit availability, wild-sightings flag
    boolean exploitInWild,
    Instant enrichedAt
) {}

record TriageVerdict(
    TriagePriority priority,
    String riskRationale,          // 2–4 sentences
    String recommendedAction,
    boolean requiresApproval,
    Instant decidedAt
) {}
enum TriagePriority { CRITICAL_IMMEDIATE, HIGH_SCHEDULED, MEDIUM_MONITORED, LOW_ACCEPTED }

record ApprovalDecision(
    String findingId,
    ApprovalStatus status,
    String analystId,
    String reason,
    Instant decidedAt
) {}
enum ApprovalStatus { PENDING, GRANTED, REJECTED }

record DriftAlert(
    String alertId,
    String description,            // human-readable drift narrative
    double baselineCriticalPct,
    double observedCriticalPct,
    Instant detectedAt
) {}

record Finding(
    String findingId,
    Optional<RawFinding> raw,
    Optional<EnrichedFinding> enriched,
    Optional<TriageVerdict> verdict,
    Optional<ApprovalDecision> approval,
    Optional<DriftAlert> driftAlert,
    FindingStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum FindingStatus {
    INGESTED, ENRICHED, TRIAGING, VERDICT_RECORDED,
    PENDING_APPROVAL, REMEDIATED, REMEDIATION_REJECTED,
    MONITORED, ACCEPTED, FAILED
}
```

Events on `FindingEntity`: `FindingIngested`, `FindingEnriched`, `TriageStarted`, `VerdictRecorded`, `ApprovalRequested`, `RemediationApproved`, `RemediationRejected`, `FindingMonitored`, `FindingAccepted`, `DriftAlertRaised`, `FindingFailed`.

Events on `ApprovalEntity`: `ApprovalCreated`, `ApprovalGranted`, `ApprovalRejected`.

Every nullable lifecycle field on the `Finding` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/findings` — body `{ cveId, cvssScore, affectedAsset, description, submittedBy }` → `{ findingId }`.
- `GET /api/findings` — list all findings, newest-first.
- `GET /api/findings/{id}` — one finding.
- `GET /api/findings/sse` — Server-Sent Events; one event per state transition.
- `POST /api/findings/{id}/approve` — body `{ analystId, reason }` → `204`.
- `POST /api/findings/{id}/reject` — body `{ analystId, reason }` → `204`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: SecOps Vulnerability Triage Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of ingested findings (priority badge + status pill + CVE id + age + drift-alert banner when active) and a right pane with the selected finding's detail — raw finding, enriched context, triage verdict, approval controls (for findings in `PENDING_APPROVAL`), and any active drift alert.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: wired on `VulnerabilityTriageAgent` via the agent's guardrail-configuration block, bound to the `before-tool-call` hook. When the agent proposes any tool call that matches the remediation-action pattern (patch deployment, firewall-rule change, service restart, credential rotation), the guardrail intercepts it before execution, records the proposed action in `ApprovalEntity`, transitions `FindingEntity` to `PENDING_APPROVAL`, and returns a structured `approval-required` response to the agent loop. The agent loop pauses; `TriageWorkflow.approvalGateStep` polls `ApprovalEntity` until `GRANTED` or `REJECTED`.
- **H1 — analyst approval gate** (`hitl`, `application`): `TriageWorkflow.approvalGateStep` pauses the workflow until `ApprovalEntity` transitions. The analyst interacts via `POST /api/findings/{id}/approve` or `POST /api/findings/{id}/reject`. On `GRANTED`, the workflow advances to `remediationStep`. On `REJECTED`, the entity transitions to `REMEDIATION_REJECTED` and the workflow ends. No automated action can circumvent this gate.
- **E1 — periodic drift evaluator** (`eval-periodic`, `drift-fairness-watch`): `DriftCheckWorkflow` runs every 6 hours. `DriftEvaluator` fetches the last 50 triaged findings from `FindingView`, computes the distribution across `TriagePriority` values, and compares against the stored baseline. If the `CRITICAL_IMMEDIATE` percentage deviates by more than 15 percentage points from baseline, or if `LOW_ACCEPTED` exceeds 60% of recent decisions, a `DriftAlertRaised` event is emitted and surfaces in the UI. The evaluator is deterministic and rule-based — no LLM call.

## 9. Agent prompts

- `VulnerabilityTriageAgent` → `prompts/vulnerability-triage-agent.md`. The single decision-making LLM. System prompt instructs it to read the enriched finding attachment, assess exploitability and asset impact, and return a `TriageVerdict` with a structured priority, risk rationale, and recommended action.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Analyst ingests the critical container-escape seed finding; within 30 s the triage verdict appears with `CRITICAL_IMMEDIATE` priority, a risk rationale, a recommended action, and the finding transitions to `PENDING_APPROVAL`.
2. **J2** — The analyst approves the remediation; the workflow advances to `REMEDIATED`; the approve/reject buttons disappear from the UI.
3. **J3** — The before-tool-call guardrail intercepts a remediation tool call the agent proposes; the finding transitions to `PENDING_APPROVAL` before any action executes; the analyst's approval is required.
4. **J4** — The drift evaluator detects that the last 50 findings skewed toward `LOW_ACCEPTED` beyond the threshold; a drift-alert banner appears on the App UI tab.
5. **J5** — A medium-severity finding skips the approval gate; it transitions directly to `MONITORED` after `VERDICT_RECORDED`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system.

```
Create a sample named secops-triage demonstrating the single-agent × governance-risk cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-governance-risk-secops-triage. Java package
io.akka.samples.secopsvulnerabilitytriageagent. Akka 3.6.0. HTTP port 9190.

Components to wire (exactly):

- 1 AutonomousAgent VulnerabilityTriageAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/vulnerability-triage-agent.md>) and
  .capability(TaskAcceptance.of(TriageTasks.TRIAGE_FINDING).maxIterationsPerTask(3)).
  The task receives the enriched finding as a task ATTACHMENT named "finding.json"
  (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the canonical
  call). Output: TriageVerdict{priority: TriagePriority, riskRationale: String,
  recommendedAction: String, requiresApproval: boolean, decidedAt: Instant}. The agent is
  configured with a before-tool-call guardrail (see G1 in eval-matrix.yaml) registered via
  the agent's guardrail-configuration block.

- 1 Workflow TriageWorkflow per findingId with four steps:
  * awaitEnrichedStep — polls FindingEntity.getFinding every 1s; on finding.enriched().isPresent()
    advances to triageStep. WorkflowSettings.stepTimeout 15s.
  * triageStep — emits TriageStarted, then calls componentClient.forAutonomousAgent(
    VulnerabilityTriageAgent.class, "triage-" + findingId).runSingleTask(
      TaskDef.instructions("Triage the attached security finding.")
        .attachment("finding.json", serializeEnrichedFinding(finding.enriched().get()))
    ) — returns a taskId, then forTask(taskId).result(TriageTasks.TRIAGE_FINDING) to fetch
    the verdict. On success calls FindingEntity.recordVerdict(verdict). If verdict
    requiresApproval, calls FindingEntity.requestApproval() and advances to approvalGateStep.
    Otherwise advances to the appropriate terminal step. WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(2).failoverTo(TriageWorkflow::error).
  * approvalGateStep — polls ApprovalEntity.getApproval every 2s up to 30 min timeout; on
    GRANTED advances to remediationStep; on REJECTED calls FindingEntity.recordRejection and
    ends. WorkflowSettings.stepTimeout 1800s.
  * remediationStep — calls FindingEntity.markRemediated(). WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 Workflow DriftCheckWorkflow — a recurring workflow triggered on a timer every 6 hours.
  Single step: driftCheckStep — invokes DriftEvaluator.evaluate(recentFindings), emits
  DriftAlertRaised on FindingEntity(id="drift-monitor") if drift detected.
  WorkflowSettings.stepTimeout 30s.

- 1 EventSourcedEntity FindingEntity (one per findingId). State Finding{findingId: String,
  raw: Optional<RawFinding>, enriched: Optional<EnrichedFinding>, verdict: Optional<TriageVerdict>,
  approval: Optional<ApprovalDecision>, driftAlert: Optional<DriftAlert>, status: FindingStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. FindingStatus enum: INGESTED, ENRICHED,
  TRIAGING, VERDICT_RECORDED, PENDING_APPROVAL, REMEDIATED, REMEDIATION_REJECTED, MONITORED,
  ACCEPTED, FAILED. Events: FindingIngested{raw}, FindingEnriched{enriched}, TriageStarted{},
  VerdictRecorded{verdict}, ApprovalRequested{}, RemediationApproved{approval},
  RemediationRejected{approval}, FindingMonitored{}, FindingAccepted{}, DriftAlertRaised{alert},
  FindingFailed{reason}. emptyState() returns Finding.initial("") with no commandContext()
  reference (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 EventSourcedEntity ApprovalEntity (one per findingId). State ApprovalDecision. Events:
  ApprovalCreated{findingId}, ApprovalGranted{analystId, reason, decidedAt}, ApprovalRejected
  {analystId, reason, decidedAt}. Commands: create, grant, reject, getApproval.

- 1 Consumer FindingEnricher subscribed to FindingEntity events; on FindingIngested: looks up
  AssetContext from in-process seeded asset registry, fetches CVE metadata from in-process
  seeded threat-intel data, builds EnrichedFinding, calls FindingEntity.attachEnriched(enriched).
  After attachEnriched lands, starts TriageWorkflow(id = "triage-" + findingId).

- 1 View FindingView with row type FindingRow (mirrors Finding). Table updater consumes
  FindingEntity events. ONE query getAllFindings: SELECT * AS findings FROM finding_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * FindingEndpoint at /api with POST /findings (body {cveId, cvssScore, affectedAsset,
    description, submittedBy}; mints findingId; calls FindingEntity.ingest; returns {findingId}),
    GET /findings (list from getAllFindings, sorted newest-first), GET /findings/{id} (one row),
    GET /findings/sse (Server-Sent Events forwarded from view's stream-updates), POST
    /findings/{id}/approve (body {analystId, reason}; calls ApprovalEntity.grant), POST
    /findings/{id}/reject (body {analystId, reason}; calls ApprovalEntity.reject), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- TriageTasks.java declaring one Task<R> constant: TRIAGE_FINDING = Task.name("Triage finding")
  .description("Read the attached enriched security finding and produce a TriageVerdict")
  .resultConformsTo(TriageVerdict.class). DO NOT skip this — the AutonomousAgent requires its
  companion Tasks class (Lesson 7).

- Domain records RawFinding, AssetContext, EnrichedFinding, TriageVerdict, TriagePriority,
  ApprovalDecision, ApprovalStatus, DriftAlert, Finding, FindingStatus.

- RemediationGuardrail.java implementing the before-tool-call hook. Identifies remediation-class
  tool calls (patch-deploy, firewall-rule-change, service-restart, credential-rotation) by
  matching tool name against a known-remediation-tools set. On match: creates an ApprovalEntity
  record, calls FindingEntity.requestApproval(), returns Guardrail.block("approval-required")
  to pause the agent loop. Non-remediation tool calls pass through unmodified.

- DriftEvaluator.java — pure deterministic logic (no LLM). Inputs: list of recent FindingRow.
  Outputs: Optional<DriftAlert>. Scoring logic: if CRITICAL_IMMEDIATE% deviates > 15 pp from
  baseline, or LOW_ACCEPTED% > 60%, return a DriftAlert; otherwise empty. Baseline computed
  from the first 20 triaged findings; stored as static state on DriftEvaluator.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9190 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/findings.jsonl with 3 seeded findings: a critical container-
  escape CVE (CVSS 9.8, internet-facing TIER1 asset), a high-severity SQL injection (CVSS 8.1,
  TIER2 asset), and a medium-severity misconfiguration (CVSS 5.3, TIER3 asset, exploit not in wild).

- src/main/resources/sample-events/asset-registry.jsonl with 5 seeded assets covering TIER1,
  TIER2, TIER3 criticality levels with varying internet-facing flags and owner teams.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (G1, H1, E1) matching Section 8 of this
  SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes reflecting security-finding data,
  decisions.authority_level = recommend-only (the agent's verdict is advisory for MEDIUM/LOW;
  approval-gated for CRITICAL/HIGH), oversight.human_in_loop = true (analyst approval gate),
  failure.failure_modes including "severity-underestimation", "false-positive-triage",
  "guardrail-bypass-attempt", "drift-undetected"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/vulnerability-triage-agent.md loaded as the agent system prompt.

- README.md at the project root as described in SPEC.md Section 1.

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no npm).
  Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left = live list
  of finding cards with priority badge + status pill + CVE id; right = selected finding detail
  with enriched context, triage verdict, approval controls, drift-alert banner).
  Browser title exactly: <title>Akka Sample: SecOps Vulnerability Triage Agent</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. VulnerabilityTriageAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion TriageTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout: awaitEnrichedStep 15s, triageStep
  60s, approvalGateStep 1800s, remediationStep 5s, driftCheckStep 30s, error 5s.
- Lesson 6: every nullable lifecycle field on the Finding row record is Optional<T>.
- Lesson 7: TriageTasks.java with TRIAGE_FINDING = Task.name(...).description(...)
  .resultConformsTo(TriageVerdict.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated model names.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9190 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words.
- Lesson 24: index.html includes the mermaid CSS overrides and themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList index.
  Exactly five <section class="tab-panel"> elements — no more.
- The single-agent invariant: exactly ONE AutonomousAgent (VulnerabilityTriageAgent). The drift
  evaluator (DriftEvaluator.java) is deterministic rule-based — NO LLM call.
- The enriched finding is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated triageStep uses TaskDef.attachment(...).
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as an external post-hoc check.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
