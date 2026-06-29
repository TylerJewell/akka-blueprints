# SPEC — security-triage-agent

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** AI Security Triage Agent.
**One-line pitch:** Report an incident; a coordinator delegates vulnerability scanning and threat-context gathering to two workers in parallel, then merges their findings into a triage report that a security officer must approve before any mitigation executes.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to two AutonomousAgents in parallel, gathers their results, and asks a third AutonomousAgent to produce a prioritised triage report. The blueprint also demonstrates a **human-in-the-loop** gate that holds the workflow at `AWAITING_APPROVAL` until a security officer submits an approve or reject decision through the REST API, and an **eval-event** that samples each triage decision for quality scoring.

## 3. User-facing flows

The user opens the App UI tab and submits an incident signal via the form.

1. The system creates an `Incident` record in `RECEIVED` and starts a `TriageWorkflow`.
2. The SecurityCoordinator decomposes the incident into two parallel work items: a vulnerability scan query for the VulnerabilityScanner, and a threat-context query for the ThreatContextAgent.
3. The workflow forks: both agents run concurrently. Each returns a typed payload.
4. The SecurityCoordinator merges the two payloads into a `TriageReport { summary, vulnerabilities, threatContext, riskLevel, mitigationPlan }`.
5. The workflow transitions the incident to `AWAITING_APPROVAL` and waits. The human-in-the-loop endpoint is open: `POST /api/incidents/{id}/approve` or `.../reject`.
6. On approval, the workflow executes the mitigation plan and transitions the incident to `MITIGATED`. On rejection, the incident moves to `REJECTED` with the officer's reason recorded.
7. If either worker times out after 60 seconds, the workflow short-circuits: the coordinator triages from whichever partial output exists and the incident enters `DEGRADED`.

An `IncidentSimulator` (TimedAction) drips a sample incident signal every 90 seconds so the App UI is not empty on first load.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `SecurityCoordinator` | `AutonomousAgent` | Decomposes the incident, merges worker findings into a triage report, rates risk level. | `TriageWorkflow` | returns typed result to workflow |
| `VulnerabilityScanner` | `AutonomousAgent` | Identifies and scores vulnerabilities from the incident signal. Seeded CVE tool returns canned results. | `TriageWorkflow` | — |
| `ThreatContextAgent` | `AutonomousAgent` | Retrieves threat-actor context and attack-pattern history for the affected asset. | `TriageWorkflow` | — |
| `TriageWorkflow` | `Workflow` | Coordinates the parallel fan-out, the merge, the approval gate, and the mitigation step. | `IncidentEndpoint`, `IncidentConsumer` | `IncidentEntity`, `ApprovalEntity` |
| `IncidentEntity` | `EventSourcedEntity` | Holds the incident's lifecycle (received → triaging → awaiting-approval → mitigated / rejected / degraded). | `TriageWorkflow` | `IncidentView` |
| `ApprovalEntity` | `EventSourcedEntity` | Tracks the human-approval gate: records the officer's identity, decision, and reason. | `IncidentEndpoint`, `TriageWorkflow` | `TriageWorkflow` (via command) |
| `IncidentQueue` | `EventSourcedEntity` | Logs each submitted incident signal for replay and audit. | `IncidentEndpoint`, `IncidentSimulator` | `IncidentConsumer` |
| `IncidentView` | `View` | List-of-incidents read model. | `IncidentEntity` events | `IncidentEndpoint` |
| `IncidentConsumer` | `Consumer` | Listens to `IncidentQueue` events and starts a `TriageWorkflow` per submission. | `IncidentQueue` events | `TriageWorkflow` |
| `IncidentSimulator` | `TimedAction` | Drips a sample incident signal every 90 s. | scheduler | `IncidentQueue` |
| `EvalSampler` | `TimedAction` | Samples one mitigated incident every 5 minutes for eval scoring; emits a `TriageEvalScored` event. | scheduler | `IncidentEntity` |
| `IncidentEndpoint` | `HttpEndpoint` | `/api/incidents/*` — submit, get, list, approve, reject, SSE. | — | `IncidentView`, `IncidentQueue`, `IncidentEntity`, `ApprovalEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record IncidentSignal(String assetId, String assetType, String signalDescription,
                      String reportedBy, Severity initialSeverity) {}

record VulnerabilityFinding(String cveId, String description, double cvssScore,
                            String affectedComponent, String source) {}

record VulnerabilityBundle(List<VulnerabilityFinding> vulnerabilities,
                           Severity aggregateSeverity, Instant scannedAt) {}

record ThreatActorContext(String actorGroup, String attackPattern, String targetProfile,
                         String historicalPrecedent, Instant contextAt) {}

record ThreatContextBundle(List<ThreatActorContext> actors, String summary, Instant gatheredAt) {}

record ScanPlan(String vulnerabilityScanQuery, String threatContextQuery) {}

record TriageReport(String summary, VulnerabilityBundle vulnerabilities,
                    ThreatContextBundle threatContext, RiskLevel riskLevel,
                    String mitigationPlan, Instant triageAt) {}

record ApprovalDecision(String officerId, String decision, Optional<String> reason,
                        Instant decidedAt) {}

record Incident(
    String incidentId,
    String assetId,
    String assetType,
    String signalDescription,
    String reportedBy,
    IncidentStatus status,
    Severity severity,
    Optional<VulnerabilityBundle> vulnerabilities,
    Optional<ThreatContextBundle> threatContext,
    Optional<TriageReport> triageReport,
    Optional<ApprovalDecision> approval,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant receivedAt,
    Optional<Instant> resolvedAt
) {}

enum IncidentStatus { RECEIVED, TRIAGING, AWAITING_APPROVAL, MITIGATED, REJECTED, DEGRADED }
enum Severity { LOW, MEDIUM, HIGH, CRITICAL }
enum RiskLevel { LOW, MODERATE, HIGH, CRITICAL }
```

### Events (on `IncidentEntity`)

`IncidentReceived`, `TriagingStarted`, `VulnerabilitiesAttached`, `ThreatContextAttached`,
`TriageReportReady`, `AwaitingApproval`, `IncidentMitigated`, `IncidentRejected`,
`IncidentDegraded`, `TriageEvalScored`.

### Events (on `IncidentQueue`)

`IncidentSubmitted { incidentId, assetId, signalDescription, reportedBy, submittedAt }`.

### Events (on `ApprovalEntity`)

`ApprovalRequested { incidentId, requestedAt }`, `ApprovalGranted { officerId, reason, decidedAt }`,
`ApprovalDenied { officerId, reason, decidedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/incidents` — body `{ assetId, assetType, signalDescription, reportedBy, initialSeverity }` → `{ incidentId }`. Starts a workflow.
- `GET /api/incidents` — list all incidents. Optional `?status=RECEIVED|TRIAGING|AWAITING_APPROVAL|MITIGATED|REJECTED|DEGRADED`.
- `GET /api/incidents/{id}` — one incident.
- `POST /api/incidents/{id}/approve` — body `{ officerId, reason? }` → approves the pending mitigation.
- `POST /api/incidents/{id}/reject` — body `{ officerId, reason }` → rejects the pending mitigation.
- `GET /api/incidents/sse` — server-sent events stream of every incident change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "AI Security Triage Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit an incident signal, live list of incidents with status pills, approve/reject buttons for incidents in `AWAITING_APPROVAL`, expand-row to see vulnerabilities + threat context + triage report + eval score.

Browser title: `<title>Akka Sample: AI Security Triage Agent</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — human-in-the-loop approval** (`application` flavor): the `TriageWorkflow` parks at `AWAITING_APPROVAL` and sends an `ApprovalRequested` event to `ApprovalEntity`. The workflow resumes only when `POST /api/incidents/{id}/approve` or `.../reject` is called. No mitigation executes without officer sign-off.
- **E1 — eval-event sampler** (`on-incident-reporter` flavor): `EvalSampler` (TimedAction) picks one `MITIGATED` incident every 5 minutes and emits a `TriageEvalScored` event with a 1–5 score and short rationale assessing triage accuracy and risk-level assignment.

## 9. Agent prompts

- `SecurityCoordinator` → `prompts/security-coordinator.md`. Decomposes the incident into parallel work items; later merges findings into the triage report.
- `VulnerabilityScanner` → `prompts/vulnerability-scanner.md`. Identifies and scores vulnerabilities; returns `VulnerabilityBundle`.
- `ThreatContextAgent` → `prompts/threat-context-agent.md`. Retrieves threat-actor context; returns `ThreatContextBundle`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit an incident; report progresses RECEIVED → TRIAGING → AWAITING_APPROVAL within 60 s; UI shows approve/reject buttons.
2. **J2** — Approve the mitigation; incident moves to MITIGATED; UI reflects the change via SSE.
3. **J3** — Reject the mitigation; incident moves to REJECTED with the officer's reason.
4. **J4** — Inject a worker timeout; incident enters DEGRADED with partial triage output.
5. **J5** — Wait for EvalSampler; a MITIGATED incident gains an evalScore on the App UI.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named security-triage-agent demonstrating the
delegation-supervisor-workers × governance-risk cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-governance-risk-security-triage-agent.
Java package io.akka.samples.aisecurityagent. Akka 3.6.0. HTTP port 9464.

Components to wire (exactly):
- 3 AutonomousAgents:
  * SecurityCoordinator — definition() with
    capability(TaskAcceptance.of(DECOMPOSE).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(TRIAGE).maxIterationsPerTask(3)).
    System prompt loaded from prompts/security-coordinator.md. Returns
    ScanPlan{vulnerabilityScanQuery, threatContextQuery} for DECOMPOSE and
    TriageReport{summary, vulnerabilities, threatContext, riskLevel,
    mitigationPlan, triageAt} for TRIAGE.
  * VulnerabilityScanner — capability(TaskAcceptance.of(SCAN).maxIterationsPerTask(3)).
    System prompt from prompts/vulnerability-scanner.md. Returns
    VulnerabilityBundle{vulnerabilities: List<VulnerabilityFinding{cveId, description,
    cvssScore, affectedComponent, source}>, aggregateSeverity, scannedAt}.
  * ThreatContextAgent — capability(TaskAcceptance.of(GATHER_CONTEXT).maxIterationsPerTask(2)).
    System prompt from prompts/threat-context-agent.md. Returns
    ThreatContextBundle{actors: List<ThreatActorContext{actorGroup, attackPattern,
    targetProfile, historicalPrecedent, contextAt}>, summary, gatheredAt}.

- 1 Workflow TriageWorkflow with steps:
  decomposeStep -> [parallel] scanStep, contextStep -> joinStep -> triageStep ->
  approvalStep (park) -> mitigateStep | rejectStep -> emitStep.
  decomposeStep calls forAutonomousAgent(SecurityCoordinator.class, DECOMPOSE).
  scanStep and contextStep run in parallel (CompletionStage zip); each governed by
  WorkflowSettings.builder().stepTimeout(TriageWorkflow::scanStep, ofSeconds(60)) and
  stepTimeout(TriageWorkflow::contextStep, ofSeconds(60)). On either timeout, transition to
  a degradeStep that calls triageStep with whichever side returned, then ends with
  IncidentDegraded.
  triageStep calls forAutonomousAgent(SecurityCoordinator.class, TRIAGE) with the merged
  inputs; give triageStep a 90s stepTimeout.
  approvalStep transitions the incident to AWAITING_APPROVAL and parks the workflow until
  ApprovalEntity delivers a resume signal (approve or reject command).
  On approval, mitigateStep executes the mitigationPlan and ends with IncidentMitigated.
  On rejection, rejectStep ends with IncidentRejected.
  WorkflowSettings is nested inside Workflow — no import.

- 2 EventSourcedEntities:
  * IncidentEntity holding state Incident{incidentId, assetId, assetType,
    signalDescription, reportedBy, IncidentStatus, Severity, Optional<VulnerabilityBundle>,
    Optional<ThreatContextBundle>, Optional<TriageReport>, Optional<ApprovalDecision>,
    Optional<String> failureReason, Optional<Integer> evalScore,
    Optional<String> evalRationale, Instant receivedAt, Optional<Instant> resolvedAt}.
    IncidentStatus enum: RECEIVED, TRIAGING, AWAITING_APPROVAL, MITIGATED, REJECTED, DEGRADED.
    Severity enum: LOW, MEDIUM, HIGH, CRITICAL. RiskLevel enum: LOW, MODERATE, HIGH, CRITICAL.
    Events: IncidentReceived, TriagingStarted, VulnerabilitiesAttached, ThreatContextAttached,
    TriageReportReady, AwaitingApproval, IncidentMitigated, IncidentRejected,
    IncidentDegraded, TriageEvalScored.
    Commands: receiveIncident, startTriaging, attachVulnerabilities, attachThreatContext,
    reportReady, awaitApproval, mitigate, reject, degrade, recordEval, getIncident.
    emptyState() returns Incident.initial("", "", "", "", null, null) with no commandContext() reference.
  * ApprovalEntity tracking the human-approval gate per incident. Events: ApprovalRequested,
    ApprovalGranted, ApprovalDenied. Commands: requestApproval, approve, deny, getApproval.

- 1 EventSourcedEntity IncidentQueue with command submitIncident(assetId, signalDescription,
  reportedBy, initialSeverity) emitting IncidentSubmitted{incidentId, assetId, signalDescription,
  reportedBy, submittedAt}.

- 1 View IncidentView with row type IncidentRow (mirrors Incident minus heavy nested payloads;
  every nullable field is Optional<T>). Table updater consumes IncidentEntity events.
  ONE query getAllIncidents SELECT * AS incidents FROM incident_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer IncidentConsumer subscribed to IncidentQueue events; on IncidentSubmitted
  starts a TriageWorkflow with the incidentId as the workflow id.

- 2 TimedActions:
  * IncidentSimulator — every 90s, reads next line from
    src/main/resources/sample-events/incident-signals.jsonl and calls
    IncidentQueue.submitIncident.
  * EvalSampler — every 5 minutes, queries IncidentView.getAllIncidents, picks the oldest
    MITIGATED incident without an evalScore, runs a 1–5 rubric judge over the triage content,
    then calls IncidentEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * IncidentEndpoint at /api with POST /incidents, GET /incidents, GET /incidents/{id},
    POST /incidents/{id}/approve, POST /incidents/{id}/reject,
    GET /incidents/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- SecurityTasks.java declaring four Task<R> constants: DECOMPOSE (ScanPlan), SCAN
  (VulnerabilityBundle), GATHER_CONTEXT (ThreatContextBundle), TRIAGE (TriageReport).
- Domain records ScanPlan, VulnerabilityFinding, VulnerabilityBundle, ThreatActorContext,
  ThreatContextBundle, TriageReport, ApprovalDecision, IncidentSignal.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9464 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/incident-signals.jsonl with 8 canned incident lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (H1 human-in-the-loop application
  flavor, E1 eval-event on-incident-reporter flavor) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function = security-triage,
  decisions.authority_level = recommend-only (human approves), data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/security-coordinator.md, prompts/vulnerability-scanner.md,
  prompts/threat-context-agent.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: AI Security Triage Agent", one-line
  pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills and approve/reject buttons for AWAITING_APPROVAL incidents). Browser title exactly:
  <title>Akka Sample: AI Security Triage Agent</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
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
  src/main/resources/mock-responses/<agent-name>.json (security-coordinator.json,
  vulnerability-scanner.json, threat-context-agent.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    security-coordinator.json — list of either ScanPlan or TriageReport objects.
      4–6 ScanPlan entries (vulnerabilityScanQuery + threatContextQuery pairs) and
      4–6 TriageReport entries (each with a 60–120 word summary, a VulnerabilityBundle
      with 2–4 findings, a ThreatContextBundle with 1–3 actors, riskLevel = HIGH,
      and a concrete mitigationPlan sentence).
    vulnerability-scanner.json — 4–6 VulnerabilityBundle entries, each with 2–4
      VulnerabilityFinding records whose cveId values follow the CVE-YYYY-NNNNN pattern,
      cvssScore between 5.0 and 9.8, and source values such as "NVD 2024", "MITRE ATT&CK".
    threat-context-agent.json — 4–6 ThreatContextBundle entries, each with 1–3
      ThreatActorContext records naming a plausible threat group (e.g., "APT-28",
      "Lazarus Group"), a MITRE ATT&CK technique, a targetProfile, and a one-sentence
      historicalPrecedent.
- A MockModelProvider.seedFor(incidentId) helper makes the selection deterministic per
  incident id so the same incident produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s workers,
  90s triage); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion SecurityTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9464 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and arrow
  labels clip.
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
