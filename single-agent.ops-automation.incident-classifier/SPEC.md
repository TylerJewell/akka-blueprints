# SPEC — incident-classifier

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** ITSM Incident Categorization Agent.
**One-line pitch:** An IT incident arrives with a short description and a long description; one AI agent reads both (passed as a task attachment, never as inline prompt text) and returns a structured `ClassificationResult` — a category, subcategory, and the affected configuration item (CI) drawn from a closed taxonomy.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the ops-automation domain. One `IncidentClassifierAgent` (AutonomousAgent) carries the entire classification decision; the surrounding components only prepare its input and audit its output. Two governance mechanisms are wired around the agent:

- A **continuous-accuracy evaluator** (`eval-periodic`) tracks category-assignment drift over time. After each classification the result is compared against the closed label vocabulary; accuracy is aggregated into a rolling window. The UI's Eval Matrix tab exposes the current window's accuracy percentage so operations staff can detect model drift before it affects SLAs.
- An **on-decision evaluator** (`eval-event`) runs immediately after every `ClassificationRecorded` event, spot-checking the result: does the category code exist in the taxonomy? does the subcategory belong under that category? does the CI name match the CI registry? A mismatch scores the result low (1–2 out of 5) and surfaces a flag on the UI card so a human can override before the downstream workflow picks it up.

The blueprint shows that the single-agent pattern supports layered audit without adding a second decision-making model.

## 3. User-facing flows

The user opens the App UI tab.

1. The user pastes an incident's **Short description** and **Long description** into the respective textareas (or clicks **Load seeded example** to fill from one of three seeded incidents: a database connection failure, a VPN authentication outage, and a file-share permission denial).
2. The user provides a **Caller ID** (free text identifying the reporter) and optionally a **Priority hint** (P1–P4; not used for classification, only recorded).
3. The user clicks **Submit incident**. The UI POSTs to `/api/incidents` and receives an `incidentId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `TAXONOMY_VALIDATED` — the taxonomy check ran and confirmed the scope of recognisable codes.
5. Within ~10–30 s, the workflow's `classifyStep` completes. The card transitions to `CLASSIFYING` then `CLASSIFICATION_RECORDED`. The classification appears: a **Category** chip, a **Subcategory** chip, an **Affected CI** label, and a short `rationale` sentence the agent produced.
6. Within ~1 s of the classification, the `evalStep` finishes. The card shows an **eval score** chip (1–5) and a one-line rationale describing whether the category and CI codes match the taxonomy.
7. The user can submit more incidents; the live list retains full history.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ClassificationEndpoint` | `HttpEndpoint` | `/api/incidents/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `IncidentEntity`, `ClassificationView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `IncidentEntity` | `EventSourcedEntity` | Per-incident lifecycle: submitted → validated → classifying → classified → evaluated. Source of truth. | `ClassificationEndpoint`, `VocabularyValidator`, `ClassificationWorkflow` | `ClassificationView` |
| `VocabularyValidator` | `Consumer` | Subscribes to `IncidentSubmitted` events; validates that the taxonomy lookup table is loaded; emits `TaxonomyValidated` back to the entity. | `IncidentEntity` events | `IncidentEntity` |
| `ClassificationWorkflow` | `Workflow` | One workflow per incident. Steps: `awaitValidatedStep` → `classifyStep` → `evalStep`. | started by `VocabularyValidator` once `TaxonomyValidated` lands | `IncidentClassifierAgent`, `IncidentEntity` |
| `IncidentClassifierAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the category taxonomy in the task instructions and the incident description fields as a task attachment; returns `ClassificationResult`. | invoked by `ClassificationWorkflow` | returns classification |
| `ClassificationView` | `View` | Read model: one row per incident for the UI. | `IncidentEntity` events | `ClassificationEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record IncidentSubmission(
    String incidentId,
    String shortDescription,
    String longDescription,
    String callerId,
    String priorityHint,         // P1|P2|P3|P4 — recorded, not used for classification
    Instant submittedAt
) {}

record TaxonomyScope(
    int categoryCount,
    int subcategoryCount,
    int ciCount,
    Instant validatedAt
) {}

record ClassificationResult(
    String category,             // e.g. "Network", "Database", "Security"
    String subcategory,          // e.g. "Connectivity", "Performance", "Authentication"
    String affectedCi,           // CI name from the registry, e.g. "PROD-DB-01"
    String rationale,            // 1–2 sentences from the agent
    Instant decidedAt
) {}

record AccuracyContribution(
    boolean categoryInTaxonomy,
    boolean subcategoryUnderCategory,
    boolean ciInRegistry,
    int score,                   // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record Incident(
    String incidentId,
    Optional<IncidentSubmission> submission,
    Optional<TaxonomyScope> scope,
    Optional<ClassificationResult> classification,
    Optional<AccuracyContribution> eval,
    IncidentStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum IncidentStatus {
    SUBMITTED, TAXONOMY_VALIDATED, CLASSIFYING, CLASSIFICATION_RECORDED, EVALUATED, FAILED
}
```

Events on `IncidentEntity`: `IncidentSubmitted`, `TaxonomyValidated`, `ClassificationStarted`, `ClassificationRecorded`, `AccuracyScored`, `ClassificationFailed`.

Every nullable lifecycle field on the `Incident` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/incidents` — body `{ shortDescription, longDescription, callerId, priorityHint }` → `{ incidentId }`.
- `GET /api/incidents` — list all incidents, newest-first.
- `GET /api/incidents/{id}` — one incident.
- `GET /api/incidents/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: ITSM Incident Categorization Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted incidents (status pill + category chip + age) and a right pane with the selected incident's detail — submission fields, taxonomy scope summary, classification result (category, subcategory, CI, rationale), and eval-score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — on-decision eval** (`eval-event`, `on-decision-eval`): runs immediately after each `ClassificationRecorded` event, as `evalStep` inside the workflow. A deterministic scorer (no LLM call) checks that `category` is in the taxonomy table, that `subcategory` belongs under that `category`, and that `affectedCi` appears in the CI registry. Computes a 1–5 score. Emits `AccuracyScored`. Score ≤ 2 flags the card in the UI for human override.
- **E2 — continuous-accuracy eval** (`eval-periodic`, `continuous-accuracy`): every `AccuracyScored` event contributes to a rolling 7-day window. The window tracks the fraction of classifications where all three fields validated (the "fully-correct" rate). The ClassificationView maintains the rolling aggregate; the Eval Matrix tab renders the current window percentage and a sparkline of daily accuracy so ops staff can detect drift.

## 9. Agent prompts

- `IncidentClassifierAgent` → `prompts/incident-classifier.md`. The single decision-making LLM. System prompt instructs it to read the attached incident description, match it against the provided category taxonomy, and return one `ClassificationResult` with a category, subcategory, and affected CI drawn from the closed lists.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the "database connection failure" seed; within 30 s the classification appears with a valid category, subcategory, CI, and an eval score chip.
2. **J2** — A classification result where the agent returns an unrecognised category code receives an eval score of 1; the UI flags the card and shows the mismatch in the rationale.
3. **J3** — The continuous-accuracy rolling window updates after each classification; submitting 5 incidents causes the Eval Matrix tab's accuracy percentage to reflect the fraction that scored ≥ 3.
4. **J4** — The mock LLM path deterministically produces one out-of-vocabulary category on every 4th incident (modulo seed), confirming the on-decision evaluator fires for that case.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named incident-classifier demonstrating the single-agent × ops-automation cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-ops-automation-incident-classifier. Java package
io.akka.samples.itsmincidentcategorizationagent. Akka 3.6.0. HTTP port 9562.

Components to wire (exactly):

- 1 AutonomousAgent IncidentClassifierAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/incident-classifier.md>) and
  .capability(TaskAcceptance.of(CLASSIFY_INCIDENT).maxIterationsPerTask(3)). The task
  receives the category taxonomy as its instruction text and the incident description fields
  as a task ATTACHMENT (NOT as inline prompt text — Akka's TaskDef.attachment(name,
  contentBytes) is the canonical call). Output: ClassificationResult{category: String,
  subcategory: String, affectedCi: String, rationale: String, decidedAt: Instant}.

- 1 Workflow ClassificationWorkflow per incidentId with three steps:
  * awaitValidatedStep — polls IncidentEntity.getIncident every 1s; on
    incident.scope().isPresent() advances to classifyStep. WorkflowSettings.stepTimeout 15s.
  * classifyStep — emits ClassificationStarted, then calls componentClient.forAutonomousAgent(
    IncidentClassifierAgent.class, "classifier-" + incidentId).runSingleTask(
      TaskDef.instructions(formatTaxonomy(taxonomyTable))
        .attachment("incident.txt", formatIncident(incident.submission).getBytes())
    ) — returns a taskId, then forTask(taskId).result(CLASSIFY_INCIDENT) to fetch the result.
    On success calls IncidentEntity.recordClassification(result). WorkflowSettings.stepTimeout
    60s with defaultStepRecovery maxRetries(2).failoverTo(ClassificationWorkflow::error).
  * evalStep — runs a deterministic rule-based AccuracyEvaluator (NOT an LLM call) over the
    recorded classification: checks that category is in the taxonomy table, that subcategory
    belongs under that category, and that affectedCi appears in the CI registry. Computes a
    1–5 score. Emits AccuracyScored{categoryInTaxonomy, subcategoryUnderCategory,
    ciInRegistry, score, rationale}. WorkflowSettings.stepTimeout 5s. error step transitions
    entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity IncidentEntity (one per incidentId). State Incident{incidentId: String,
  submission: Optional<IncidentSubmission>, scope: Optional<TaxonomyScope>,
  classification: Optional<ClassificationResult>, eval: Optional<AccuracyContribution>,
  status: IncidentStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  IncidentStatus enum: SUBMITTED, TAXONOMY_VALIDATED, CLASSIFYING, CLASSIFICATION_RECORDED,
  EVALUATED, FAILED. Events: IncidentSubmitted{submission}, TaxonomyValidated{scope},
  ClassificationStarted{}, ClassificationRecorded{classification}, AccuracyScored{eval},
  ClassificationFailed{reason}. Commands: submit, markValidated, markClassifying,
  recordClassification, recordAccuracy, fail, getIncident. emptyState() returns
  Incident.initial("") with no commandContext() reference (Lesson 3). Every Optional<T>
  field uses Optional.empty() in initial state and Optional.of(...) inside event-appliers.

- 1 Consumer VocabularyValidator subscribed to IncidentEntity events; on IncidentSubmitted
  reads the in-process taxonomy table (loaded from src/main/resources/taxonomy/taxonomy.json
  at startup), counts categories, subcategories, and CI entries, builds TaxonomyScope, then
  calls IncidentEntity.markValidated(scope). After markValidated lands, the same Consumer
  starts a ClassificationWorkflow with id = "classification-" + incidentId.

- 1 View ClassificationView with row type IncidentRow (mirrors Incident). Table updater
  consumes IncidentEntity events. TWO queries:
    getAllIncidents: SELECT * AS incidents FROM classification_view (newest-first via
      createdAt DESC).
    getAccuracyWindow: SELECT * AS contributions FROM classification_view WHERE
      eval IS NOT NULL ORDER BY createdAt DESC LIMIT 50.
  No WHERE status filter on getAllIncidents — Akka cannot auto-index enum columns (Lesson 2);
  caller filters client-side.

- 2 HttpEndpoints:
  * ClassificationEndpoint at /api with POST /incidents (body
    {shortDescription, longDescription, callerId, priorityHint}; mints incidentId; calls
    IncidentEntity.submit; returns {incidentId}), GET /incidents (list from getAllIncidents,
    sorted newest-first), GET /incidents/{id} (one row), GET /incidents/sse (Server-Sent
    Events forwarded from the view's stream-updates), and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- IncidentTasks.java declaring one Task<R> constant: CLASSIFY_INCIDENT = Task
  .name("Classify incident").description("Read the attached incident description and produce
  a ClassificationResult with category, subcategory, and affectedCi drawn from the provided
  taxonomy").resultConformsTo(ClassificationResult.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records IncidentSubmission, TaxonomyScope, ClassificationResult,
  AccuracyContribution, Incident, IncidentStatus.

- AccuracyEvaluator.java — pure deterministic logic (no LLM). Inputs: ClassificationResult
  and the loaded TaxonomyTable. Outputs: AccuracyContribution. Scoring rubric: +2 if
  category is in taxonomy, +2 if subcategory is under that category, +1 if CI is in
  registry = total 5. Scoring documented in Javadoc on the class.

- TaxonomyTable.java — singleton loaded from src/main/resources/taxonomy/taxonomy.json at
  startup. Provides methods: containsCategory(String), subcategoryBelongsTo(String cat,
  String sub), containsCi(String). Loaded once, immutable, never from LLM output.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9562 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/taxonomy/taxonomy.json — the seeded category taxonomy. Structure:
  { "categories": [ { "code": "Network", "subcategories": ["Connectivity","DNS","VPN","Firewall"],
  "ciCodes": ["PROD-FW-01","PROD-SW-01","CORP-VPN-01"] }, ... ] }. Seed with 6 categories:
  Network, Database, Security, Storage, Application, Identity. Each category has 4–6
  subcategories and 3–5 CI codes. Total: ~30 subcategories, ~25 CIs.

- src/main/resources/sample-events/incidents.jsonl — 3 seeded incident descriptions:
  (1) "Database connection failure" — longDescription describes intermittent connection
    timeouts to PROD-DB-01 from the application tier;
  (2) "VPN authentication outage" — users unable to authenticate to CORP-VPN-01 after a
    certificate renewal;
  (3) "File-share permission denial" — a team cannot write to a shared folder on PROD-NAS-01
    after an AD group policy change.
  Each has a callerId, a priorityHint, and known correct category+subcategory+CI so J1/J4
  are reproducible.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (E1, E2) matching the mechanisms in
  Section 8. Matching simplified_view list.

- risk-survey.yaml at the project root with purpose.primary_function = it-incident-routing,
  decisions.authority_level = automated-with-override (classification is applied
  automatically but a human can override before the downstream workflow acts),
  oversight.human_on_loop = true, failure.failure_modes including "wrong-category",
  "wrong-subcategory", "unknown-ci", "accuracy-drift"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/incident-classifier.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: ITSM Incident Categorization Agent",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of incident cards with status pill, category chip, age; right = selected-incident
  detail with submission fields, taxonomy scope summary, classification result with category/
  subcategory/CI chips and rationale, and eval-score chip).
  Browser title exactly:
  <title>Akka Sample: ITSM Incident Categorization Agent</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with per-task
  dispatch on the Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json, picks one entry pseudo-randomly per call
  (seedFor(incidentId)), and deserialises into the task's typed return.
- Per-task mock-response shapes:
    classify-incident.json — 9 ClassificationResult entries covering all 6 categories
      with varying subcategories and CIs. Plus 1 deliberately out-of-vocabulary entry
      (category = "UnknownService", subcategory = "Generic", ci = "UNKNOWN-CI-99") and 1
      entry with a valid category but wrong subcategory — both trigger eval score ≤ 2.
      The mock selects the out-of-vocabulary entry on every 4th incident (modulo seed)
      so J4 is reproducible.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. IncidentClassifierAgent
  extends akka.javasdk.agent.autonomous.AutonomousAgent. The companion IncidentTasks.java
  MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (classifyStep 60s,
  awaitValidatedStep 15s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Incident row record is Optional<T>.
- Lesson 7: IncidentTasks.java with CLASSIFY_INCIDENT = Task.name(...).description(...)
  .resultConformsTo(ClassificationResult.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9562 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (IncidentClassifierAgent).
  The on-decision evaluator and continuous-accuracy evaluator are both deterministic and
  rule-based (AccuracyEvaluator.java) and do NOT make LLM calls.
- The incident description is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify classifyStep uses TaskDef.attachment(...).
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
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
