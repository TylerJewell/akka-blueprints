# SPEC — incident-management

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Incident Management.
**One-line pitch:** A classifier agent categorizes an incoming incident report and hands the investigation off to an infrastructure or application specialist that owns it end-to-end, with context enrichment, a before-tool-call guardrail on every external write, and an inline eval scoring every routing decision.

## 2. What this blueprint demonstrates

The **handoff-routing** coordination pattern — one classifier agent decides *who* should own the investigation, then transfers the same incident identity to a downstream specialist agent that produces the final remediation action. The downstream agent is responsible for the whole investigation; the classifier does not narrate or summarise. Two governance mechanisms are layered on top:

- A **before-tool-call guardrail** runs inside the workflow before any specialist-initiated write action reaches an external system. The specialist proposes a `RemediationAction` (e.g., restart a service, scale up a deployment, open a change record). The guardrail checks the proposed action against a policy rubric (no production writes outside a change window, no destructive actions on critical-tier hosts, no restarts during peak traffic hours) and blocks the action when it violates, transitioning the incident to `BLOCKED` for human review.
- An **on-incident-reporter eval** fires every time the classifier agent emits a routing decision. A `RoutingJudge` agent grades the decision against the enriched payload on a 1–5 rubric. The score and rationale are written back to the incident entity and surfaced in the UI.

The pattern is a textbook fan-out-of-one: the workflow branches on the classifier's category, and only the chosen specialist is invoked. The other specialist sees no traffic for that incident.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live incident list. Every incident displays its category chip, status pill, routing score, and (if resolved) the published remediation summary.
2. `IncidentFeeder` (TimedAction) ticks every 30 s and inserts a new canned report from `sample-events/incidents.jsonl` into `IncidentQueue`.
3. For each new report: `ContextEnricher` (Consumer) attaches environment metadata, registers an `IncidentEntity`, and starts an `IncidentWorkflow`.
4. The workflow calls `ClassifierAgent`, gets a `ClassificationDecision { category, confidence, reason }`, and emits `IncidentClassified` on the entity.
5. Branch on `category`:
   - `INFRASTRUCTURE` → workflow calls `InfraSpecialist` with the `INVESTIGATE` task and waits for the typed `RemediationPlan` result.
   - `APPLICATION` → workflow calls `AppSpecialist` with the same `INVESTIGATE` task.
   - `AMBIGUOUS` → workflow emits `IncidentEscalated`; ends.
6. The specialist's proposed `RemediationPlan` passes through the before-tool-call guardrail. If accepted, `RemediationPublished` is emitted (terminal `RESOLVED`). If rejected, `RemediationBlocked` is emitted (terminal `BLOCKED`) with the violation list.
7. Independent of the workflow, `RoutingEvalScorer` (Consumer) listens for `IncidentClassified` events, calls `RoutingJudge`, and writes `RoutingScored { score, rationale }` back to the incident entity.
8. The user can click any incident card and see the enriched report, the classification reason, the routing score, the chosen specialist, the proposed remediation (or blocked plan + violations), and the published action summary.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `IncidentFeeder` | `TimedAction` | Drips simulated incident reports into `IncidentQueue` every 30 s. | scheduler | `IncidentQueue` |
| `IncidentQueue` | `EventSourcedEntity` | Append-only audit log of every inbound report (`InboundReportReceived`). | `IncidentFeeder`, `IncidentEndpoint` | `ContextEnricher` |
| `ContextEnricher` | `Consumer` | Subscribes to `IncidentQueue` events; attaches env metadata; registers `IncidentEntity`; starts an `IncidentWorkflow`. | `IncidentQueue` events | `IncidentEntity`, `IncidentWorkflow` |
| `ClassifierAgent` | `Agent` (typed, not autonomous) | Classifies an `EnrichedReport` into `INFRASTRUCTURE` / `APPLICATION` / `AMBIGUOUS` with confidence + reason. | invoked by `IncidentWorkflow` | returns `ClassificationDecision` |
| `InfraSpecialist` | `AutonomousAgent` | Owns the `INVESTIGATE` task for infrastructure incidents. Returns typed `RemediationPlan`. | invoked by `IncidentWorkflow` | returns `RemediationPlan` |
| `AppSpecialist` | `AutonomousAgent` | Owns the `INVESTIGATE` task for application incidents. Returns typed `RemediationPlan`. | invoked by `IncidentWorkflow` | returns `RemediationPlan` |
| `RoutingJudge` | `Agent` (typed) | Grades a classification decision against the enriched payload. Returns `RoutingScore { score 1–5, rationale }`. | invoked by `RoutingEvalScorer` | returns `RoutingScore` |
| `ToolCallGuardrail` | `Agent` (typed) | Before-tool-call guardrail: checks a proposed `RemediationPlan` against the ops policy rubric. Returns `GuardrailVerdict { allowed, violations }`. | invoked by `IncidentWorkflow` | returns `GuardrailVerdict` |
| `IncidentWorkflow` | `Workflow` | Per-incident orchestration: classify → branch → investigate → guardrail → publish. | `ContextEnricher` (start) | `IncidentEntity` |
| `IncidentEntity` | `EventSourcedEntity` | Per-incident lifecycle. | `IncidentWorkflow`, `RoutingEvalScorer` | `IncidentView` |
| `IncidentView` | `View` | Read-model row per incident. | `IncidentEntity` events | `IncidentEndpoint` |
| `RoutingEvalScorer` | `Consumer` | Subscribes to `IncidentEntity` events; on `IncidentClassified` invokes `RoutingJudge` and writes `RoutingScored` back. | `IncidentEntity` events | `IncidentEntity` |
| `IncidentEndpoint` | `HttpEndpoint` | `/api/incidents/*` — list, get, manual submit, manual unblock, SSE; `/api/metadata/*`. | — | `IncidentView`, `IncidentEntity`, `IncidentQueue` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record InboundReport(
    String incidentId,
    String reportedBy,
    String alertSource,        // "pagerduty" | "datadog" | "manual"
    String title,
    String description,
    Instant reportedAt
) {}

record EnrichedReport(
    String incidentId,
    String title,
    String description,
    String hostGroup,          // e.g. "prod-us-east-1" | "staging-eu-west-1"
    String serviceTier,        // "critical" | "standard" | "dev"
    List<String> tags          // e.g. ["network","latency","deploy"]
) {}

enum IncidentCategory { INFRASTRUCTURE, APPLICATION, AMBIGUOUS }

record ClassificationDecision(
    IncidentCategory category,
    String confidence,         // "high" | "medium" | "low"
    String reason              // one short sentence
) {}

enum RemediationAction { RESTART_SERVICE, SCALE_DEPLOYMENT, ROLLBACK_DEPLOY,
                         OPEN_CHANGE_RECORD, PAGE_ON_CALL, INFO_PROVIDED }

record RemediationPlan(
    String summary,
    String details,
    RemediationAction action,
    String specialistTag,      // "infra" | "app"
    Instant proposedAt
) {}

record GuardrailVerdict(
    boolean allowed,
    List<String> violations,   // empty when allowed
    String rubricVersion
) {}

record RoutingScore(
    int score,                 // 1..5
    String rationale,
    Instant scoredAt
) {}

record Incident(
    String incidentId,
    InboundReport report,
    Optional<EnrichedReport> enriched,
    Optional<ClassificationDecision> classification,
    Optional<RemediationPlan> remediation,
    Optional<GuardrailVerdict> guardrail,
    Optional<RoutingScore> routingScore,
    Optional<String> escalationReason,
    IncidentStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum IncidentStatus {
    RECEIVED,
    ENRICHED,
    CLASSIFIED,
    ROUTED_INFRA,
    ROUTED_APP,
    PLAN_DRAFTED,
    BLOCKED,
    RESOLVED,
    ESCALATED
}
```

Events on `IncidentEntity`: `IncidentRegistered`, `IncidentEnriched`, `IncidentClassified`, `IncidentRouted`, `PlanDrafted`, `GuardrailVerdictAttached`, `RemediationPublished`, `RemediationBlocked`, `IncidentEscalated`, `RoutingScored`.

Events on `IncidentQueue`: `InboundReportReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/incidents` — list all incidents (newest-first), optional `?category=INFRASTRUCTURE|APPLICATION|AMBIGUOUS&status=…` filtered client-side.
- `GET /api/incidents/{id}` — one incident.
- `POST /api/incidents` — manually submit a report (body `InboundReport` minus `incidentId` and `reportedAt`); server assigns both.
- `POST /api/incidents/{id}/unblock` — body `{ decidedBy, note }` — operator override; transitions `BLOCKED` to `RESOLVED` if the operator approves the blocked plan.
- `GET /api/incidents/sse` — Server-Sent Events for every incident change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Incident Management</title>`.

The App UI tab is a three-pane layout: **left** is the incident list (status pill + category chip + score chip), **centre** is the selected incident's enriched report + classification decision + score, **right** is the chosen specialist's remediation plan + guardrail verdict + published summary (or violations + Unblock button when `BLOCKED`).

Tab switching is attribute-based (`data-tab` / `data-panel`); no zombie panels in the DOM. The Architecture tab's mermaid diagrams carry the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels are not clipped.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** on `InfraSpecialist` and `AppSpecialist`: checks every proposed `RemediationPlan` against a rubric (no production writes outside a change window, no destructive actions on critical-tier hosts, no restarts during peak traffic hours, no echoing of raw alert tokens in a public summary). Blocking — a violation puts the incident in `BLOCKED` for human review.
- **E1 — on-incident-reporter eval** (`eval-event`, on the classification decision): `RoutingEvalScorer` (Consumer) listens for `IncidentClassified` events and calls `RoutingJudge` to produce a 1–5 score with a one-sentence rationale. Non-blocking — the score is metadata, not a gate; persistent low scores would be surfaced as a system-level alert in a deployed setting.

## 9. Agent prompts

- `ClassifierAgent` → `prompts/classifier-agent.md`. Typed classifier; returns one of `INFRASTRUCTURE`, `APPLICATION`, `AMBIGUOUS`; defaults to `AMBIGUOUS` under uncertainty.
- `InfraSpecialist` → `prompts/infra-specialist.md`. Owns the `INVESTIGATE` task for infrastructure incidents. Never performs destructive actions outside its authority; routes outside its scope to `PAGE_ON_CALL`.
- `AppSpecialist` → `prompts/app-specialist.md`. Owns the `INVESTIGATE` task for application incidents. Cites a runbook id when one applies; never invents one.
- `RoutingJudge` → `prompts/routing-judge.md`. Grades a classification decision against a 1–5 rubric with one-sentence rationale.
- `ToolCallGuardrail` → `prompts/tool-call-guardrail.md`. Returns a `GuardrailVerdict { allowed, violations }`. Conservative — borderline plans are blocked.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Feeder drips an infrastructure-flavoured incident → classified `INFRASTRUCTURE` → remediated by `InfraSpecialist` → guardrail passes → published.
2. **J2** — Feeder drips an application incident → classified `APPLICATION` → remediated by `AppSpecialist` → guardrail passes → published.
3. **J3** — An ambiguous incident classifies as `AMBIGUOUS` and lands in `ESCALATED` without any specialist invocation.
4. **J4** — A plan that proposes a production restart during peak hours is blocked; the incident lands in `BLOCKED` with the violation listed; the operator can either unblock or leave it.
5. **J5** — Every classified incident carries a `RoutingScore` (1–5) and rationale within ~10 s of the classification decision.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named incident-management demonstrating the handoff-routing × ops-automation cell.
Runs out of the box (in-process simulated inbound stream; no real alerting integration).
Maven group io.akka.samples. Maven artifact handoff-routing-ops-automation-incident-mgmt. Java package
io.akka.samples.incidentmanagement. Akka 3.6.0. HTTP port 9635.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) ClassifierAgent — classifier. System prompt loaded from
  prompts/classifier-agent.md. Input: EnrichedReport{incidentId, title, description,
  hostGroup, serviceTier, tags: List<String>}. Output: ClassificationDecision{category:
  IncidentCategory (INFRASTRUCTURE/APPLICATION/AMBIGUOUS), confidence: "high"|"medium"|"low",
  reason: String}. Defaults to AMBIGUOUS under uncertainty.

- 1 AutonomousAgent InfraSpecialist — definition() with capability(TaskAcceptance.of(INVESTIGATE)
  .maxIterationsPerTask(3)). System prompt from prompts/infra-specialist.md. Input:
  EnrichedReport + ClassificationDecision. Output: RemediationPlan{summary, details,
  action: RemediationAction, specialistTag = "infra", proposedAt}. Never performs destructive
  actions on critical-tier hosts without explicit authority; sets action=PAGE_ON_CALL when
  outside its scope.

- 1 AutonomousAgent AppSpecialist — definition() with capability(TaskAcceptance.of(INVESTIGATE)
  .maxIterationsPerTask(3)). System prompt from prompts/app-specialist.md. Same input shape;
  specialistTag = "app".

- 1 Agent (typed) RoutingJudge — judge. System prompt from prompts/routing-judge.md. Input:
  EnrichedReport + ClassificationDecision. Output: RoutingScore{score: int 1–5, rationale:
  String, scoredAt: Instant}.

- 1 Agent (typed) ToolCallGuardrail — typed rubric check. System prompt from
  prompts/tool-call-guardrail.md. Input: EnrichedReport + RemediationPlan. Output:
  GuardrailVerdict{allowed: boolean, violations: List<String>, rubricVersion: String}.
  Used by IncidentWorkflow before publishing; blocking.

- 1 Workflow IncidentWorkflow per incidentId. Steps:
    classifyStep -> routeStep -> {infraStep | appStep | escalateStep}
                -> guardrailStep -> publishStep
  classifyStep calls componentClient.forAgent().inSession(incidentId).method(ClassifierAgent::classify)
    .invoke(enriched). On success emits IncidentClassified via IncidentEntity.recordClassification.
  routeStep branches on ClassificationDecision.category:
    INFRASTRUCTURE -> proceed to infraStep (emits IncidentRouted{INFRASTRUCTURE})
    APPLICATION -> proceed to appStep (emits IncidentRouted{APPLICATION})
    AMBIGUOUS -> escalateStep (emits IncidentEscalated; terminates).
  infraStep / appStep call forAutonomousAgent(<Specialist>.class, incidentId)
    .runSingleTask(TaskDef.instructions(buildPrompt(enriched, classification))) returning a
    taskId, then forTask(taskId).result(IncidentTasks.INVESTIGATE) to block on the typed
    RemediationPlan. On success emits PlanDrafted.
  guardrailStep calls forAgent(...).method(ToolCallGuardrail::check).invoke(enriched, plan).
    On verdict.allowed=true proceed to publishStep (emits RemediationPublished, terminal
    RESOLVED). On verdict.allowed=false emit RemediationBlocked (terminal BLOCKED) and end.
  Override settings() with stepTimeout(Duration.ofSeconds(20)) on classifyStep and
    guardrailStep, stepTimeout(Duration.ofSeconds(60)) on infraStep, appStep, and
    publishStep. defaultStepRecovery(maxRetries(2).failoverTo(IncidentWorkflow::error)).

- 2 EventSourcedEntities:
    * IncidentQueue — append-only audit log. Command receive(InboundReport) emits
      InboundReportReceived{report}. No mutable state beyond a counter; commands are
      idempotent on report.incidentId.
    * IncidentEntity (one per incidentId) — full per-incident lifecycle. State
      Incident{incidentId, report: InboundReport, Optional<EnrichedReport> enriched,
      Optional<ClassificationDecision> classification, Optional<RemediationPlan> remediation,
      Optional<GuardrailVerdict> guardrail, Optional<RoutingScore> routingScore,
      Optional<String> escalationReason, IncidentStatus status, Instant createdAt,
      Optional<Instant> finishedAt}. IncidentStatus enum: RECEIVED, ENRICHED, CLASSIFIED,
      ROUTED_INFRA, ROUTED_APP, PLAN_DRAFTED, BLOCKED, RESOLVED, ESCALATED.
      Events: IncidentRegistered, IncidentEnriched, IncidentClassified, IncidentRouted,
      PlanDrafted, GuardrailVerdictAttached, RemediationPublished, RemediationBlocked,
      IncidentEscalated, RoutingScored. Commands: registerReport, attachEnriched,
      recordClassification, recordRouting, recordPlan, recordGuardrailVerdict, publish,
      block, escalate, recordRoutingScore, unblock, getIncident. emptyState() returns
      Incident.initial("") with no commandContext() reference.

- 2 Consumers:
    * ContextEnricher subscribed to IncidentQueue events; for each InboundReportReceived
      builds an EnrichedReport by attaching hostGroup (derived from alertSource + a
      deterministic mapping for the sample), serviceTier (from a static tier registry keyed
      on alertSource), and tags (extracted via keyword scan of title + description for
      "network", "latency", "cpu", "memory", "deploy", "timeout", "error", "disk"),
      calls IncidentEntity.registerReport then attachEnriched for the incidentId; then starts
      an IncidentWorkflow with incidentId as the workflow id.
    * RoutingEvalScorer subscribed to IncidentEntity events; on IncidentClassified invokes
      RoutingJudge.score(enriched, decision) and calls IncidentEntity.recordRoutingScore(
      incidentId, score). On any other event type, no-op. Use componentClient — do NOT call
      the agent from a TimedAction.

- 1 View IncidentView with row type IncidentRow (mirrors Incident; uses Optional<T> for every
  nullable lifecycle field per Lesson 6). Table updater consumes IncidentEntity events.
  ONE query getAllIncidents SELECT * AS incidents FROM incident_view. No WHERE category or
  WHERE status filter (Akka cannot auto-index enum columns) — filter client-side in callers.

- 1 TimedAction IncidentFeeder — every 30s, reads next line from
  src/main/resources/sample-events/incidents.jsonl (loops at EOF) and calls
  IncidentQueue.receive with a fresh incidentId (UUID).

- 2 HttpEndpoints:
    * IncidentEndpoint at /api with GET /incidents (list from IncidentView.getAllIncidents,
      filter client-side by ?category and ?status query params), GET /incidents/{id},
      POST /incidents (body InboundReport minus incidentId/reportedAt — server assigns),
      POST /incidents/{id}/unblock (body {decidedBy, note} — operator override:
      publishes the blocked plan as RESOLVED with an audit note),
      GET /incidents/sse (serverSentEventsForView over getAllIncidents), and three
      /api/metadata/{readme,risk-survey,eval-matrix} endpoints serving the YAML/MD files
      from src/main/resources/metadata/.
    * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- IncidentTasks.java declaring the task constants: INVESTIGATE (resultConformsTo
  RemediationPlan.class, description "Investigate the incident end-to-end and return a
  typed RemediationPlan").
- Domain records InboundReport, EnrichedReport, ClassificationDecision, RemediationPlan,
  GuardrailVerdict, RoutingScore, and the Incident entity state.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9635 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/incidents.jsonl with 9 canned lines (3 INFRASTRUCTURE-
  flavoured, 3 APPLICATION-flavoured, 2 AMBIGUOUS-flavoured, 1 designed to trip the guardrail
  with a production restart proposed during peak hours).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: G1 guardrail before-tool-call
  policy-rubric, E1 eval-event on-incident-reporter. Matching simplified_view list. No
  regulation_anchors (community-content sample).
- risk-survey.yaml at the project root with purpose.primary_function = incident-management,
  data.data_classes.pii = false, decisions.authority_level = draft-only,
  oversight.human_in_loop = false (the system publishes without HITL by default — only
  blocked plans wait for a human), failure.failure_modes including
  "wrong-category-routing", "inappropriate-remediation-action",
  "destructive-action-on-critical-host", "peak-hours-write"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/classifier-agent.md, prompts/infra-specialist.md, prompts/app-specialist.md,
  prompts/routing-judge.md, prompts/tool-call-guardrail.md loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: Incident Management", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a three-column layout
  (left = incident list with category chip + status pill + score chip; centre = enriched
  report + classification block; right = specialist plan + guardrail verdict + published
  summary or violations + Unblock button). Browser title exactly:
  <title>Akka Sample: Incident Management</title>. No subtitle on the Overview tab.

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
  pseudo-randomly per call (seeded by incidentId so reruns are deterministic),
  and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    classifier-agent.json — 12 ClassificationDecision entries spanning
      INFRASTRUCTURE (network timeouts, disk saturation, CPU spikes),
      APPLICATION (deploy-triggered error spikes, latency regression,
      exception storms), and AMBIGUOUS (mixed signals, very short alerts,
      off-hours noise). Confidence + a one-sentence reason on each.
    infra-specialist.json — 8 RemediationPlan entries: 5 with action
      RESTART_SERVICE or SCALE_DEPLOYMENT (well within authority), 1 with
      action PAGE_ON_CALL (outside scope), 1 with action OPEN_CHANGE_RECORD,
      1 designed to trip the guardrail (proposes a critical-tier host restart
      during peak hours) so guardrail tests have material.
    app-specialist.json — 8 RemediationPlan entries: 5 with INFO_PROVIDED
      or ROLLBACK_DEPLOY, 1 with OPEN_CHANGE_RECORD, 1 with PAGE_ON_CALL,
      1 designed to trip the guardrail (sends a raw alert token in the public
      summary — "alert-token-echo" violation).
    routing-judge.json — 10 RoutingScore entries, score 1–5, one-sentence
      rationale matching the rubric (category-correctness / confidence-
      calibration / reason-quality).
    tool-call-guardrail.json — 10 GuardrailVerdict entries. 7 with
      allowed=true and empty violations. 3 with allowed=false and one
      violation each ("peak-hours-restart", "critical-host-no-authority",
      "alert-token-echo"). The mock should match the rubric-tripping entries
      above when called for the same incidentId.
- A MockModelProvider.seedFor(incidentId) helper makes per-incident selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- (Lesson 1) AutonomousAgent is never silently downgraded to Agent.
  InfraSpecialist and AppSpecialist both extend
  akka.javasdk.agent.autonomous.AutonomousAgent and declare definition().
- (Lesson 4) Workflow step timeouts overridden via settings(): classifyStep 20s,
  guardrailStep 20s, infraStep / appStep / publishStep 60s each.
- (Lesson 6) Every nullable lifecycle field on Incident is Optional<T>. The
  IncidentView row type uses the same Optional wrapping.
- (Lesson 7) IncidentTasks.java declares the INVESTIGATE Task<RemediationPlan> constant.
  Both specialists' definition().capability(TaskAcceptance.of(INVESTIGATE)...)
  reference it.
- (Lesson 8) Model names verified against current lineup: claude-sonnet-4-6,
  gpt-4o, gemini-2.5-flash.
- (Lesson 9) Run command is "/akka:build" everywhere. No "mvn akka:run".
- (Lesson 10) Port 9635 in application.conf; not 9000.
- (Lesson 11) No source.platform string anywhere user-facing.
- (Lesson 12) Static UI fits in 1080px content column with no horizontal scroll.
- (Lesson 13) Integration tier label is "Runs out of the box" — never T1.
- (Lesson 23) No competitor brand names in any user-facing text.
- (Lesson 24) static-resources/index.html includes the mermaid CSS overrides
  AND theme variables (state-diagram label colour white, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc).
- (Lesson 25) API key sourcing follows the five-option protocol above.
- (Lesson 26) Tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. No "hidden" zombie panels.
- The ContextEnricher runs INSIDE a Consumer before any LLM call — not inside
  an Agent's prompt and not after the LLM has seen the raw report.
- The RoutingEvalScorer Consumer reacts to IncidentClassified events and calls
  RoutingJudge via componentClient.forAgent(). It does NOT modify the workflow
  flow — the eval is out-of-band metadata.
- The guardrail step happens BEFORE RemediationPublished. A blocked plan
  never reaches the UI as published — only as a "blocked plan + violations"
  surface for the operator.
- No forbidden words in user-facing text: shape, minimal, smaller, complex,
  Akka SDK in narrative, T1/T2/T3/T4, deferred, use, use, marketing
  tone, competitor brand names.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local key-source reference written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
