# SPEC — medical-triage-delegation

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Medical Agent Delegation.
**One-line pitch:** Submit a medical question; a triage coordinator delegates it to three specialist agents in parallel, applies a safety guardrail, and synthesises a patient-safe response.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to three AutonomousAgents in parallel, gathers their results, and asks a fourth AutonomousAgent to synthesise a unified response. The blueprint also demonstrates a **special-category data sanitizer** that strips personally identifiable health data before any agent receives it, a **before-agent-response safety guardrail** that blocks responses containing diagnosis claims, and a **human-on-the-loop (hotl)** compliance event that flags cases for clinical review.

## 3. User-facing flows

The user opens the App UI tab and submits a medical question via the form.

1. The system creates a `MedicalCase` record in `TRIAGING` and starts a `TriageWorkflow`.
2. The sanitizer strips PII and special-category health markers from the question before any agent receives it.
3. The TriageCoordinator classifies the question and decomposes it into three specialist queries: a symptoms query, a medications query, and a care-pathway query.
4. The workflow forks: all three specialists run concurrently. Each returns a typed payload.
5. The TriageCoordinator merges the three payloads into a `SynthesisedResponse { summary, symptomsAssessment, medicationGuidance, careRecommendation, guardrailVerdict }`.
6. A before-agent-response safety guardrail vets the response; if it contains diagnosis claims, the case moves to `BLOCKED`. Otherwise, the case moves to `RESPONDED`.
7. A hotl event is emitted so a clinical compliance reviewer can inspect the response before it is acted upon.
8. If any specialist times out after 60 seconds, the workflow short-circuits: it asks the TriageCoordinator to synthesise from whichever specialists returned, and the case enters `DEGRADED`.

A `CaseSimulator` (TimedAction) drips a sample question every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TriageCoordinator` | `AutonomousAgent` | Classifies the question, decomposes it into specialist queries, synthesises the merged response, runs the safety guardrail. | `TriageWorkflow` | returns typed result to workflow |
| `SymptomsSpecialist` | `AutonomousAgent` | Interprets symptom descriptions; returns a `SymptomsAssessment`. | `TriageWorkflow` | — |
| `MedicationsSpecialist` | `AutonomousAgent` | Checks drug interactions, contraindications, dosage guidance; returns a `MedicationGuidance`. | `TriageWorkflow` | — |
| `CarePathwaySpecialist` | `AutonomousAgent` | Recommends the appropriate care pathway; returns a `CareRecommendation`. | `TriageWorkflow` | — |
| `TriageWorkflow` | `Workflow` | Coordinates the parallel fan-out, synthesis, guardrail, and hotl notification. | `TriageEndpoint`, `CaseRequestConsumer` | `CaseEntity` |
| `CaseEntity` | `EventSourcedEntity` | Holds the case lifecycle (triaging → in-review → responded / degraded / blocked). | `TriageWorkflow` | `CaseView` |
| `CaseQueue` | `EventSourcedEntity` | Logs each submitted question for replay and audit. | `TriageEndpoint`, `CaseSimulator` | `CaseRequestConsumer` |
| `CaseView` | `View` | List-of-cases read model. | `CaseEntity` events | `TriageEndpoint` |
| `CaseRequestConsumer` | `Consumer` | Listens to `CaseQueue` events and starts a workflow per submission. | `CaseQueue` events | `TriageWorkflow` |
| `CaseSimulator` | `TimedAction` | Drips a sample question every 60 s. | scheduler | `CaseQueue` |
| `ComplianceSampler` | `TimedAction` | Samples one RESPONDED case every 5 minutes; emits a `ComplianceChecked` event with a safety score. | scheduler | `CaseEntity` |
| `TriageEndpoint` | `HttpEndpoint` | `/api/triage/*` — submit, get, list, SSE. | — | `CaseView`, `CaseQueue`, `CaseEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record QuestionRequest(String question, String submittedBy) {}

record SymptomsAssessment(
    List<String> identifiedSymptoms,
    String differentialSummary,
    String urgencyIndicator,
    Instant assessedAt
) {}

record MedicationGuidance(
    List<String> relevantMedications,
    List<String> interactions,
    List<String> contraindications,
    Instant guidanceAt
) {}

record CareRecommendation(
    String recommendedPathway,
    String rationale,
    List<String> nextSteps,
    Instant recommendedAt
) {}

record SpecialistQueries(
    String symptomsQuery,
    String medicationsQuery,
    String carePathwayQuery
) {}

record SynthesisedResponse(
    String summary,
    SymptomsAssessment symptomsAssessment,
    MedicationGuidance medicationGuidance,
    CareRecommendation careRecommendation,
    String guardrailVerdict,
    Instant synthesisedAt
) {}

record MedicalCase(
    String caseId,
    String question,
    CaseStatus status,
    Optional<SymptomsAssessment> symptomsAssessment,
    Optional<MedicationGuidance> medicationGuidance,
    Optional<CareRecommendation> careRecommendation,
    Optional<SynthesisedResponse> response,
    Optional<String> failureReason,
    Optional<Integer> complianceScore,
    Optional<String> complianceRationale,
    boolean hotlPending,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum CaseStatus { TRIAGING, IN_REVIEW, RESPONDED, DEGRADED, BLOCKED }
```

### Events (on `CaseEntity`)

`CaseCreated`, `SymptomsAssessed`, `MedicationChecked`, `CarePathwaySet`, `CaseResponded`, `CaseDegraded`, `CaseBlocked`, `HotlFlagged`, `ComplianceChecked`.

### Events (on `CaseQueue`)

`QuestionSubmitted { caseId, question, submittedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/triage` — body `{ question }` → `{ caseId }`. Starts a workflow.
- `GET /api/triage` — list all cases. Optional `?status=TRIAGING|IN_REVIEW|RESPONDED|DEGRADED|BLOCKED`.
- `GET /api/triage/{id}` — one case.
- `GET /api/triage/sse` — server-sent events stream of every case change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Medical Agent Delegation"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a question, live list of cases with status pills, expand-row to see symptoms assessment, medication guidance, care recommendation, synthesised summary, hotl flag, and compliance score.

Browser title: `<title>Akka Sample: Medical Agent Delegation</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — special-category sanitizer** (`special-category` on inbound question): strips PII and GDPR special-category health identifiers from the raw question before passing it to any agent. Non-blocking pre-processing.
- **G1 — safety guardrail** (`before-agent-response` on `TriageCoordinator`): vets the synthesised response for diagnosis claims, prescriptive medication advice, and emergency-care errors. Blocking. Failure → `BLOCKED`.
- **H1 — human-on-the-loop compliance review** (`live-compliance-review` hotl): after every `RESPONDED` case, emits a `HotlFlagged` event so a clinical compliance reviewer can inspect the response. Non-blocking.

## 9. Agent prompts

- `TriageCoordinator` → `prompts/triage-coordinator.md`. Classifies the question; decomposes it into specialist queries; later synthesises results into the final response.
- `SymptomsSpecialist` → `prompts/symptoms-specialist.md`. Interprets symptoms; returns `SymptomsAssessment`.
- `MedicationsSpecialist` → `prompts/medications-specialist.md`. Checks drug interactions and contraindications; returns `MedicationGuidance`.
- `CarePathwaySpecialist` → `prompts/care-pathway-specialist.md`. Recommends care pathway; returns `CareRecommendation`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a medical question; case progresses TRIAGING → IN_REVIEW → RESPONDED within 90 s; UI reflects each transition via SSE.
2. **J2** — Inject a specialist timeout (set `SymptomsSpecialist` timeout to 1 s); case enters DEGRADED with whichever partial outputs came back; summary notes missing coverage.
3. **J3** — Inject a guardrail failure (TriageCoordinator returns a response with an explicit diagnosis claim); case enters BLOCKED.
4. **J4** — After a successful response, the case row shows `hotlPending: true`; compliance reviewer acknowledges; flag clears.
5. **J5** — Wait for `ComplianceSampler`; the case gains a `complianceScore` and `complianceRationale`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named medical-triage-delegation demonstrating the
delegation-supervisor-workers × healthcare cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-healthcare-medical-triage-delegation.
Java package io.akka.samples.medicalagentdelegation. Akka 3.6.0. HTTP port 9155.

Components to wire (exactly):
- 4 AutonomousAgents:
  * TriageCoordinator — definition() with capability(TaskAcceptance.of(CLASSIFY).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(SYNTHESISE_RESPONSE).maxIterationsPerTask(3)). System prompt
    loaded from prompts/triage-coordinator.md. Returns SpecialistQueries{symptomsQuery,
    medicationsQuery, carePathwayQuery} for CLASSIFY and SynthesisedResponse{summary,
    symptomsAssessment, medicationGuidance, careRecommendation, guardrailVerdict,
    synthesisedAt} for SYNTHESISE_RESPONSE.
  * SymptomsSpecialist — capability(TaskAcceptance.of(ASSESS_SYMPTOMS).maxIterationsPerTask(3)).
    System prompt from prompts/symptoms-specialist.md. Returns SymptomsAssessment{identifiedSymptoms:
    List<String>, differentialSummary, urgencyIndicator, assessedAt}.
  * MedicationsSpecialist — capability(TaskAcceptance.of(CHECK_MEDICATIONS).maxIterationsPerTask(3)).
    System prompt from prompts/medications-specialist.md. Returns MedicationGuidance{relevantMedications:
    List<String>, interactions: List<String>, contraindications: List<String>, guidanceAt}.
  * CarePathwaySpecialist — capability(TaskAcceptance.of(RECOMMEND_PATHWAY).maxIterationsPerTask(2)).
    System prompt from prompts/care-pathway-specialist.md. Returns CareRecommendation{recommendedPathway,
    rationale, nextSteps: List<String>, recommendedAt}.

- 1 Workflow TriageWorkflow with steps:
  sanitizeStep -> classifyStep -> [parallel] symptomsStep, medicationsStep, carePathwayStep ->
  joinStep -> synthesiseStep -> guardrailStep -> hotlStep -> emitStep.
  sanitizeStep runs a deterministic PII/special-category stripper over the raw question before
  passing a sanitised version to all downstream agents.
  classifyStep calls forAutonomousAgent(TriageCoordinator.class, CLASSIFY).
  symptomsStep, medicationsStep, carePathwayStep run in parallel (CompletionStage zip of three);
  each governed by WorkflowSettings.builder().stepTimeout(TriageWorkflow::symptomsStep, ofSeconds(60))
  and matching timeouts for medicationsStep and carePathwayStep. On any timeout, transition to a
  degradeStep that calls synthesiseStep with whichever specialists returned, then ends with
  CaseDegraded.
  synthesiseStep calls forAutonomousAgent(TriageCoordinator.class, SYNTHESISE_RESPONSE) with the
  merged inputs; give synthesiseStep a 90s stepTimeout.
  guardrailStep runs the deterministic vetter + LLM judge on the synthesised content; on failure,
  end with CaseBlocked.
  hotlStep emits HotlFlagged on CaseEntity and sets hotlPending = true.
  WorkflowSettings is nested inside Workflow — no import.

- 1 EventSourcedEntity CaseEntity holding state MedicalCase{caseId, question, CaseStatus,
  Optional<SymptomsAssessment> symptomsAssessment, Optional<MedicationGuidance> medicationGuidance,
  Optional<CareRecommendation> careRecommendation, Optional<SynthesisedResponse> response,
  Optional<String> failureReason, Optional<Integer> complianceScore, Optional<String>
  complianceRationale, boolean hotlPending, Instant createdAt, Optional<Instant> finishedAt}.
  CaseStatus enum: TRIAGING, IN_REVIEW, RESPONDED, DEGRADED, BLOCKED. Events: CaseCreated,
  SymptomsAssessed, MedicationChecked, CarePathwaySet, CaseResponded, CaseDegraded, CaseBlocked,
  HotlFlagged, ComplianceChecked. Commands: createCase, recordSymptomsAssessment,
  recordMedicationGuidance, recordCarePathway, respond, degrade, block, flagHotl,
  acknowledgeHotl, recordCompliance, getCase.
  emptyState() returns MedicalCase.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity CaseQueue with command enqueueQuestion(question, submittedBy) emitting
  QuestionSubmitted{caseId, question, submittedBy, submittedAt}.

- 1 View CaseView with row type MedicalCaseRow (mirrors MedicalCase minus heavy nested payloads;
  every nullable field is Optional<T>). Table updater consumes CaseEntity events. ONE query
  getAllCases SELECT * AS cases FROM case_view. No WHERE status filter (Akka cannot auto-index
  enum columns) — caller filters client-side.

- 1 Consumer CaseRequestConsumer subscribed to CaseQueue events; on QuestionSubmitted starts a
  TriageWorkflow with the caseId as the workflow id.

- 2 TimedActions:
  * CaseSimulator — every 60s, reads next line from
    src/main/resources/sample-events/medical-questions.jsonl and calls CaseQueue.enqueueQuestion.
  * ComplianceSampler — every 5 minutes, queries CaseView.getAllCases, picks the oldest RESPONDED
    case without a complianceScore, runs a 1–5 safety rubric judge over the synthesised response,
    then calls CaseEntity.recordCompliance(score, rationale).

- 2 HttpEndpoints:
  * TriageEndpoint at /api with POST /triage, GET /triage, GET /triage/{id}, GET /triage/sse, and
    the /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- MedicalTriageTasks.java declaring five Task<R> constants: CLASSIFY (SpecialistQueries),
  ASSESS_SYMPTOMS (SymptomsAssessment), CHECK_MEDICATIONS (MedicationGuidance),
  RECOMMEND_PATHWAY (CareRecommendation), SYNTHESISE_RESPONSE (SynthesisedResponse).
- Domain records QuestionRequest, SpecialistQueries, SymptomsAssessment, MedicationGuidance,
  CareRecommendation, SynthesisedResponse.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9155 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/medical-questions.jsonl with 8 canned question lines
  covering common non-emergency presentations (headache with fever, persistent cough, mild
  chest tightness, skin rash, digestive discomfort, joint pain, fatigue, sleep difficulty).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (S1 sanitizer special-category,
  G1 guardrail before-agent-response, H1 hotl live-compliance-review) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector = healthcare,
  decisions.authority_level = recommend-only, data.data_classes.pii = true (health data),
  capabilities.clinical_decision_support = false (informational only); deployer fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/triage-coordinator.md, prompts/symptoms-specialist.md,
  prompts/medications-specialist.md, prompts/care-pathway-specialist.md loaded at agent
  startup as system prompts.
- README.md at the project root: title "Akka Sample: Medical Agent Delegation", one-line
  pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills). Browser title exactly: <title>Akka Sample: Medical Agent Delegation</title>. No
  subtitle on the Overview tab.

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
  src/main/resources/mock-responses/<agent-name>.json (triage-coordinator.json,
  symptoms-specialist.json, medications-specialist.json, care-pathway-specialist.json),
  picks one entry pseudo-randomly per call, and deserialises it into the agent's
  typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    triage-coordinator.json — list of either SpecialistQueries or SynthesisedResponse objects.
      4–6 SpecialistQueries entries (symptomsQuery + medicationsQuery + carePathwayQuery triples)
      and 4–6 SynthesisedResponse entries (each with an 80–150 word summary, a SymptomsAssessment,
      a MedicationGuidance, a CareRecommendation, guardrailVerdict = "ok").
    symptoms-specialist.json — 4–6 SymptomsAssessment entries, each with 2–5 identifiedSymptoms,
      a one-sentence differentialSummary, and an urgencyIndicator of "low", "moderate", or "high".
    medications-specialist.json — 4–6 MedicationGuidance entries, each with 1–4 relevantMedications,
      0–2 interactions, and 0–2 contraindications.
    care-pathway-specialist.json — 4–6 CareRecommendation entries covering pathways: "self-care",
      "pharmacist", "GP appointment", "urgent care", "emergency services"; each with a one-sentence
      rationale and 2–4 nextSteps.
- A MockModelProvider.seedFor(caseId) helper makes the selection deterministic per case id
  so the same case produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s specialists,
  90s synthesis); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion MedicalTriageTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9155 in application.conf.
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
- Parallel workflow steps use CompletionStage zip of three, NOT sequential calls.
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
