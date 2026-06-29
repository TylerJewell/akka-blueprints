# SPEC — cyber-guardian-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Cyber Guardian Agent.
**One-line pitch:** A background monitor ingests infrastructure telemetry, classifies threat signals in real time, automatically isolates CRITICAL incidents, and emits a structured eval event for every closed incident.

## 2. What this blueprint demonstrates

The **continuous-monitor** coordination pattern wired with two governance mechanisms layered on top of two AI primitives (`ThreatClassifierAgent` and `SafetyHaltAgent`). Specifically:

- An **on-incident eval event** is emitted once for every closed incident — feeding an auditable trail that downstream eval pipelines can consume. The event captures the classification, severity verdict, halt decision, and playbook outcome in one structured record.
- An **automatic safety halt** fires the moment a CRITICAL severity is confirmed: `SafetyHaltAgent` issues an isolation directive (simulated) and the `IncidentWorkflow` records the halt before any human is consulted. The system does not wait for approval to halt — it waits for a human to *lift* the halt.

The result is a system where high-severity incidents are contained before a human reads the alert, and every incident — regardless of severity — produces an auditable eval record.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live threat feed: every received signal, its classification, severity pill, and current lifecycle status.
2. `TelemetryPoller` (TimedAction) ticks every 10 s and inserts a new simulated `ThreatSignal` into `TelemetryQueue`. (A `RequestSimulator` style — drips canned signals with varying severities.)
3. For each new signal: `ThreatClassifierAgent` classifies it into a `ThreatAssessment` with severity, category, and a one-sentence reasoning line.
4. If severity is CRITICAL: `SafetyHaltAgent` issues an isolation directive, the incident status becomes `HALTED`, and `RemediationAdvisorAgent` then produces a playbook.
5. If severity is HIGH: `RemediationAdvisorAgent` produces a remediation playbook; no automatic halt.
6. If severity is LOW or MEDIUM: the incident closes directly after classification; no playbook needed.
7. On close (any severity), `EvalEventEmitter` emits an `IncidentEvalEvent` capturing the full assessment.
8. The operator can lift a halt via the UI, transitioning the incident from `HALTED` to `REMEDIATION` then `CLOSED`.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TelemetryPoller` | `TimedAction` | Drips simulated threat signals into `TelemetryQueue` every 10 s. | scheduler | `TelemetryQueue` |
| `TelemetryQueue` | `EventSourcedEntity` | Append-only audit log of `SignalReceived` events. | `TelemetryPoller`, `ThreatEndpoint` | `ThreatSignalConsumer` |
| `ThreatSignalConsumer` | `Consumer` | Reads `SignalReceived` events, normalises source fields, and starts an `IncidentWorkflow` per signal. | `TelemetryQueue` events | `IncidentEntity`, `IncidentWorkflow` |
| `ThreatClassifierAgent` | `Agent` (typed, not autonomous) | Scores a normalised signal into severity + category + reasoning. | invoked by Workflow | returns ThreatAssessment |
| `RemediationAdvisorAgent` | `AutonomousAgent` | Produces a step-by-step remediation playbook for HIGH/CRITICAL incidents. | invoked by Workflow | returns RemediationPlaybook |
| `SafetyHaltAgent` | `AutonomousAgent` | Issues an isolation directive for CRITICAL incidents. | invoked by Workflow | returns HaltDirective |
| `IncidentWorkflow` | `Workflow` | Per-signal orchestration: classify → branch on severity → (CRITICAL) halt → remediation → close → emit eval. | `ThreatSignalConsumer` (one workflow per `SignalReceived`) | `IncidentEntity` |
| `IncidentEntity` | `EventSourcedEntity` | Lifecycle per incident: received → classified → halted/remediation → closed. | `IncidentWorkflow` | `IncidentView` |
| `IncidentView` | `View` | Read-model row per incident for the UI. | `IncidentEntity` events | `ThreatEndpoint` |
| `EvalEventEmitter` | `TimedAction` | Every 5 min, picks CLOSED incidents without an eval record; emits `IncidentEvalEvent`. | scheduler | `IncidentEntity` |
| `ThreatEndpoint` | `HttpEndpoint` | `/api/threats/*` — list, get, lift-halt, SSE. | — | `IncidentView`, `IncidentEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record ThreatSignal(String signalId, String sourceHost, String category,
                    String rawPayload, Instant receivedAt) {}

record ThreatAssessment(Severity severity, String category, String reasoning,
                        double confidenceScore) {}
enum Severity { LOW, MEDIUM, HIGH, CRITICAL }

record RemediationPlaybook(List<String> steps, String estimatedEffort,
                           Instant generatedAt) {}

record HaltDirective(String signalId, String isolatedHost, String justification,
                     Instant issuedAt) {}

record LiftDecision(String liftedBy, String reason, Instant liftedAt) {}

record EvalEventPayload(String signalId, Severity severity, String category,
                        boolean haltIssued, boolean playbookGenerated,
                        double confidenceScore, Instant closedAt) {}

record IncidentState(
    String signalId,
    ThreatSignal signal,
    Optional<ThreatAssessment> assessment,
    Optional<HaltDirective> halt,
    Optional<LiftDecision> liftDecision,
    Optional<RemediationPlaybook> playbook,
    Optional<EvalEventPayload> evalEvent,
    IncidentStatus status,
    Instant createdAt,
    Optional<Instant> closedAt
) {}

enum IncidentStatus {
    RECEIVED, CLASSIFIED, HALTED, REMEDIATION, CLOSED, IGNORED
}
```

Events on `IncidentEntity`: `SignalReceived`, `IncidentClassified`, `HaltIssued`, `HaltLifted`, `PlaybookGenerated`, `IncidentClosed`, `EvalEventEmitted`.

Events on `TelemetryQueue`: `SignalReceived` (audit log of raw signals).

See `reference/data-model.md`.

## 6. API contract

- `GET /api/threats` — list all incidents. Optional `?status=…` or `?severity=…`.
- `GET /api/threats/{id}` — one incident.
- `POST /api/threats/{id}/lift-halt` — body `{ liftedBy, reason }` → transitions HALTED to REMEDIATION.
- `GET /api/threats/sse` — Server-Sent Events for every incident state change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,signal-categories,survey-template}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Cyber Guardian Agent</title>`.

App UI tab shows the **live threat feed** — a two-column layout. Left: incident list sorted newest-first with severity pill, category chip, and status indicator. Right: selected incident detail with assessment reasoning, halt directive (if issued), playbook steps, and (for HALTED incidents) a Lift Halt button.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — on-incident eval event** (`eval-event`, `on-incident-reporter`, applied by `EvalEventEmitter`): emits a structured `IncidentEvalEvent` for every CLOSED incident carrying severity, category, halt decision, playbook flag, and confidence score.
- **H1 — automatic safety halt** (`halt`, `automatic-safety-halt`, applied in `IncidentWorkflow` via `SafetyHaltAgent`): for CRITICAL-severity assessments, the workflow calls `SafetyHaltAgent` and records a `HaltIssued` event before any other action. No human approval is required to halt; human approval is required to lift.

## 9. Agent prompts

- `ThreatClassifierAgent` → `prompts/threat-classifier.md`. Typed classifier. Always returns a `ThreatAssessment` with exactly one `Severity` value.
- `RemediationAdvisorAgent` → `prompts/remediation-advisor.md`. AutonomousAgent producing a `RemediationPlaybook` for HIGH/CRITICAL incidents.
- `SafetyHaltAgent` → `prompts/safety-halt.md`. AutonomousAgent that produces a `HaltDirective` for CRITICAL incidents. It never lifts its own halt.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a CRITICAL signal; it is classified, halted, and a playbook generated — all within 30 s of arrival.
2. **J2** — Operator clicks Lift Halt; incident transitions to REMEDIATION then CLOSED; eval event emitted.
3. **J3** — Simulator drips a LOW signal; classified; no halt issued; incident closes directly; eval event emitted.
4. **J4** — `EvalEventEmitter` tick produces `IncidentEvalEvent` records visible in the UI within 5 minutes of close.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named cyber-guardian-agent demonstrating the continuous-monitor ×
governance-risk cell. Runs out of the box (in-memory threat feed; no real SIEM
integration). Maven group io.akka.samples, artifact
continuous-monitor-governance-risk-cyber-guardian. Java package
io.akka.samples.cyberguardianagent. HTTP port 9181.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) ThreatClassifierAgent — severity classifier. System
  prompt loaded from prompts/threat-classifier.md. Input: ThreatSignal{signalId,
  sourceHost, category, rawPayload, receivedAt}. Output: ThreatAssessment{severity:
  Severity (enum LOW/MEDIUM/HIGH/CRITICAL), category: String, reasoning: String,
  confidenceScore: double 0.0–1.0}. Defaults to HIGH under uncertainty.

- 1 AutonomousAgent RemediationAdvisorAgent — definition() with
  capability(TaskAcceptance.of(REMEDIATION).maxIterationsPerTask(4)). System prompt from
  prompts/remediation-advisor.md. Input: ThreatAssessment + ThreatSignal. Output:
  RemediationPlaybook{steps: List<String>, estimatedEffort: String, generatedAt: Instant}.
  Invoked for HIGH and CRITICAL severity only.

- 1 AutonomousAgent SafetyHaltAgent — definition() with
  capability(TaskAcceptance.of(HALT).maxIterationsPerTask(2)). System prompt from
  prompts/safety-halt.md. Input: ThreatAssessment + ThreatSignal. Output:
  HaltDirective{signalId, isolatedHost, justification, issuedAt}. Invoked for CRITICAL
  severity only. The agent NEVER lifts its own directive.

- 1 Workflow IncidentWorkflow per signal with steps:
  classifyStep -> severity branch:
    CRITICAL -> haltStep -> remediationStep -> closeStep;
    HIGH     -> remediationStep -> closeStep;
    MEDIUM   -> closeStep;
    LOW      -> ignoreStep (emits IncidentClosed with status=IGNORED).
  classifyStep wraps ThreatClassifierAgent with
  WorkflowSettings.builder().stepTimeout(Duration.ofSeconds(15)).
  haltStep wraps SafetyHaltAgent with stepTimeout 20s.
  remediationStep wraps RemediationAdvisorAgent with stepTimeout 45s.
  closeStep emits IncidentClosed. On any step timeout, escalate to CRITICAL path.

- 2 EventSourcedEntities:
  * TelemetryQueue — append-only audit log. Command receive(ThreatSignal) emits
    SignalReceived{signal}.
  * IncidentEntity (one per signalId) — full lifecycle. State IncidentState{signalId,
    signal: ThreatSignal{signalId, sourceHost, category, rawPayload, receivedAt},
    Optional<ThreatAssessment> assessment (with severity: Severity, category, reasoning,
    confidenceScore: double), Optional<HaltDirective> halt (with signalId, isolatedHost,
    justification, issuedAt), Optional<LiftDecision> liftDecision (with liftedBy, reason,
    liftedAt), Optional<RemediationPlaybook> playbook (with steps: List<String>,
    estimatedEffort, generatedAt), Optional<EvalEventPayload> evalEvent,
    IncidentStatus status, Instant createdAt, Optional<Instant> closedAt}. IncidentStatus
    enum: RECEIVED, CLASSIFIED, HALTED, REMEDIATION, CLOSED, IGNORED. Events:
    SignalReceived, IncidentClassified, HaltIssued, HaltLifted, PlaybookGenerated,
    IncidentClosed, EvalEventEmitted. Commands: registerSignal, attachAssessment,
    recordHalt, liftHalt, attachPlaybook, closeIncident, markIgnored, recordEvalEvent,
    getIncident. emptyState() returns IncidentState.initial("", null) without
    commandContext() reference.

- 1 Consumer ThreatSignalConsumer subscribed to TelemetryQueue events; for each
  SignalReceived, calls IncidentEntity.registerSignal and starts an IncidentWorkflow
  with signalId as the workflow id.

- 1 View IncidentView with row type IncidentRow (mirrors IncidentState minus raw
  rawPayload). Table updater consumes IncidentEntity events. ONE query getAllIncidents
  SELECT * AS incidents FROM incident_view. No WHERE filter — caller filters client-side.

- 2 TimedActions:
  * TelemetryPoller — every 10s, reads next line from
    src/main/resources/sample-events/threat-signals.jsonl and calls
    TelemetryQueue.receive.
  * EvalEventEmitter — every 5 minutes, queries IncidentView.getAllIncidents, picks up
    to 10 CLOSED or IGNORED incidents without an evalEvent (oldest-first), builds an
    EvalEventPayload per incident, and calls IncidentEntity.recordEvalEvent per incident.

- 2 HttpEndpoints:
  * ThreatEndpoint at /api with GET /threats, GET /threats/{id},
    POST /threats/{id}/lift-halt (body {liftedBy, reason}), GET /threats/sse, and
    three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/. lift-halt writes HaltLifted + transitions to REMEDIATION.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- IncidentTasks.java declaring three Task<R> constants: CLASSIFY (ThreatAssessment),
  REMEDIATION (RemediationPlaybook), HALT (HaltDirective).
- Domain records ThreatSignal, ThreatAssessment, RemediationPlaybook, HaltDirective,
  LiftDecision, EvalEventPayload.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9181 and the three model-provider blocks
  (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini gemini-2.5-flash)
  reading the canonical env vars.
- src/main/resources/sample-events/threat-signals.jsonl with 10 canned signal lines
  covering CRITICAL (ransomware, privilege-escalation), HIGH (lateral-movement,
  data-exfil attempt), MEDIUM (port-scan, failed-auth burst), and LOW (noise,
  benign probe) categories.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: E1 eval-event on-incident-reporter,
  H1 halt automatic-safety-halt. Matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root with sector=infrastructure-security,
  decisions.authority_level=automatic-for-halt+advisory-for-remediation,
  oversight.human_in_loop=true (halt-lift requires human), failure.failure_modes
  including "false-positive-halt" and "missed-critical-signal"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/threat-classifier.md, prompts/remediation-advisor.md, prompts/safety-halt.md
  loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: Cyber Guardian Agent",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar with the App UI tab using a two-column
  layout (left = live threat list with severity pills and category chips; right = selected
  incident detail with assessment reasoning, halt directive box, playbook steps, and
  Lift Halt button for HALTED incidents). Browser title exactly:
  <title>Akka Sample: Cyber Guardian Agent</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent.
        Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory; gone
        when the session ends.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured
  key reference if it does not resolve at runtime. The message must not echo any
  captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java implementing the ModelProvider interface with a
  per-agent dispatch. Each branch reads src/main/resources/mock-responses/<agent>.json,
  picks one entry pseudo-randomly per call, and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    threat-classifier.json — 10–12 ThreatAssessment entries spanning all four
      Severity levels. CRITICAL entries: ransomware-detected, privilege-escalation.
      HIGH: lateral-movement, data-exfil-attempt, C2-beacon. MEDIUM: port-scan,
      failed-auth-burst. LOW: benign-probe, noise. Each entry has confidenceScore
      0.6–1.0 and a one-sentence reasoning. Defaults to HIGH under ambiguity.
    remediation-advisor.json — 4–6 RemediationPlaybook entries with 4–7 step arrays
      covering isolation, credential rotation, patch application, forensic preservation.
      estimatedEffort uses strings like "2-4h", "30m", "1 business day".
    safety-halt.json — 4–6 HaltDirective entries with realistic isolatedHost names
      (e.g., "prod-db-01.internal", "k8s-node-07.us-east") and justification sentences.
- A MockModelProvider.seedFor(signalId) helper makes per-signal selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Optional<T> for every nullable field on a View row record.
- WorkflowSettings is nested inside Workflow — no import needed.
- emptyState() never calls commandContext().
- AutonomousAgent never silently downgraded to Agent.
- ThreatSignalConsumer runs INSIDE a Consumer before any LLM call — not inside an
  Agent's prompt.
- SafetyHaltAgent NEVER lifts its own directive; only the ThreatEndpoint lift-halt
  endpoint can emit HaltLifted.
- The generated static-resources/index.html must include the mermaid CSS overrides AND
  theme variables from Lesson 24 (state-diagram label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc).
- Tab switching in static-resources/index.html MUST match by data-tab / data-panel
  attribute, NEVER by NodeList index (Lesson 26). No "hidden" zombie panels in the DOM.
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var export block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK
  in narrative, T1/T2/T3/T4, marketing tone, competitor brand names.
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
