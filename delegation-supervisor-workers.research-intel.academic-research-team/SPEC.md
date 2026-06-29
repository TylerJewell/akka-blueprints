# SPEC — academic-research-team

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Academic Research Team.
**One-line pitch:** Submit a research query; a coordinator delegates publication discovery to a Paper Scout and trend analysis to a Domain Analyst in parallel, then synthesises one academic research digest.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to two AutonomousAgents in parallel, gathers their results, and asks a third AutonomousAgent to synthesise a unified digest. The blueprint also demonstrates a **citation guardrail** that vets the user-facing digest before it is returned, and **eval-event** governance that samples the coordinator's synthesis decision for a quality score.

## 3. User-facing flows

The user opens the App UI tab and submits a research query via the form.

1. The system creates a `ResearchDigest` record in `QUEUED` and starts a `ResearchWorkflow`.
2. The Coordinator decomposes the query into two parallel work items: a publication scouting directive for the PaperScout, and an interpretation question for the DomainAnalyst.
3. The workflow forks: both agents run concurrently. Each returns a typed payload.
4. The Coordinator merges the two payloads into a `SynthesisedDigest { summary, publications, trends, guardrailVerdict }`.
5. A citation guardrail vets the synthesised digest; if it fails, the digest moves to `BLOCKED`. Otherwise the digest moves to `SYNTHESISED`.
6. If either worker times out after 60 seconds, the workflow short-circuits: it asks the Coordinator to synthesise from whichever side returned, and the digest enters `DEGRADED`.

A `QuerySimulator` (TimedAction) drips a sample research query every 60 seconds so the App UI is non-empty on first load.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `LiteratureCoordinator` | `AutonomousAgent` | Decomposes the query, synthesises the merged digest, runs the citation guardrail. | `ResearchWorkflow` | returns typed result to workflow |
| `PaperScout` | `AutonomousAgent` | Discovers relevant publications for the scouting directive. Seeded "publication-search tool" returns canned results. | `ResearchWorkflow` | — |
| `DomainAnalyst` | `AutonomousAgent` | Identifies emerging trends and research gaps from the interpretation question. | `ResearchWorkflow` | — |
| `ResearchWorkflow` | `Workflow` | Coordinates the parallel fan-out, the synthesis, the citation guardrail. | `DigestEndpoint`, `QueryRequestConsumer` | `ResearchDigestEntity` |
| `ResearchDigestEntity` | `EventSourcedEntity` | Holds the digest's lifecycle (queued → scanning → synthesised / degraded / blocked). | `ResearchWorkflow` | `DigestView` |
| `QueryQueue` | `EventSourcedEntity` | Logs each submitted query for replay and audit. | `DigestEndpoint`, `QuerySimulator` | `QueryRequestConsumer` |
| `DigestView` | `View` | List-of-digests read model. | `ResearchDigestEntity` events | `DigestEndpoint` |
| `QueryRequestConsumer` | `Consumer` | Listens to `QueryQueue` events and starts a workflow per submission. | `QueryQueue` events | `ResearchWorkflow` |
| `QuerySimulator` | `TimedAction` | Drips a sample research query every 60 s. | scheduler | `QueryQueue` |
| `EvalSampler` | `TimedAction` | Samples one synthesised digest every 5 minutes for eval scoring; emits a `DigestEvalScored` event. | scheduler | `ResearchDigestEntity` |
| `DigestEndpoint` | `HttpEndpoint` | `/api/digests/*` — submit, get, list, SSE. | — | `DigestView`, `QueryQueue`, `ResearchDigestEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record QueryRequest(String query, String requestedBy) {}

record PublicationBundle(List<Publication> publications, Instant scoutedAt) {}
record Publication(String title, String doi, String venue, String abstract_) {}

record TrendReport(String thesis, List<String> emergingAreas, List<String> gaps, Instant analysedAt) {}

record ScoutingPlan(String scoutingDirective, String interpretationQuestion) {}

record SynthesisedDigest(String summary, PublicationBundle publications, TrendReport trends,
                         String guardrailVerdict, Instant synthesisedAt) {}

record ResearchDigest(
    String digestId,
    String query,
    DigestStatus status,
    Optional<PublicationBundle> publications,
    Optional<TrendReport> trends,
    Optional<SynthesisedDigest> synthesised,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum DigestStatus { QUEUED, SCANNING, SYNTHESISED, DEGRADED, BLOCKED }
```

### Events (on `ResearchDigestEntity`)

`DigestCreated`, `PublicationsAttached`, `TrendsAttached`, `DigestSynthesised`, `DigestDegraded`, `DigestBlocked`, `DigestEvalScored`.

### Events (on `QueryQueue`)

`QuerySubmitted { digestId, query, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/digests` — body `{ query }` → `{ digestId }`. Starts a workflow.
- `GET /api/digests` — list all digests. Optional `?status=QUEUED|SCANNING|SYNTHESISED|DEGRADED|BLOCKED`.
- `GET /api/digests/{id}` — one digest.
- `GET /api/digests/sse` — server-sent events stream of every digest change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Academic Research Team"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a research query, live list of digests with status pills, expand-row to see publications, trend report, synthesised summary, and eval score.

Browser title: `<title>Akka Sample: Academic Research Team</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — citation guardrail** (`before-agent-response` on `LiteratureCoordinator`): vets the synthesised digest for fabricated DOIs and unverifiable publication references. Blocking. Failure → `BLOCKED`.
- **E1 — eval-event sampler** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one synthesised digest every 5 minutes and emits a `DigestEvalScored` event with a 1–5 score and a short rationale.

## 9. Agent prompts

- `LiteratureCoordinator` → `prompts/literature-coordinator.md`. Decomposes the query into a scouting directive and interpretation question; later synthesises results into the final digest.
- `PaperScout` → `prompts/paper-scout.md`. Discovers publications; returns `PublicationBundle`.
- `DomainAnalyst` → `prompts/domain-analyst.md`. Identifies trends and gaps; returns `TrendReport`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a research query; digest progresses QUEUED → SCANNING → SYNTHESISED within 60 s; UI reflects each transition via SSE.
2. **J2** — Inject a worker timeout (set `PaperScout` timeout to 1 s); digest enters DEGRADED with whichever partial output came back.
3. **J3** — Inject a citation guardrail failure (Coordinator returns a digest with a fabricated DOI); digest enters BLOCKED.
4. **J4** — Wait after a successful synthesis; the digest's row in the UI shows an eval score.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named academic-research-team demonstrating the
delegation-supervisor-workers × research-intel cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-research-intel-academic-research-team.
Java package io.akka.samples.academicresearch. Akka 3.6.0. HTTP port 9686.

Components to wire (exactly):
- 3 AutonomousAgents:
  * LiteratureCoordinator — definition() with capability(TaskAcceptance.of(DECOMPOSE).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(SYNTHESISE).maxIterationsPerTask(3)). System prompt loaded
    from prompts/literature-coordinator.md. Returns ScoutingPlan{scoutingDirective, interpretationQuestion}
    for DECOMPOSE and SynthesisedDigest{summary, publications, trends, guardrailVerdict, synthesisedAt}
    for SYNTHESISE.
  * PaperScout — capability(TaskAcceptance.of(SCOUT).maxIterationsPerTask(3)). System prompt
    from prompts/paper-scout.md. Returns PublicationBundle{publications: List<Publication{title,
    doi, venue, abstract_}>, scoutedAt}.
  * DomainAnalyst — capability(TaskAcceptance.of(ANALYSE).maxIterationsPerTask(2)). System prompt
    from prompts/domain-analyst.md. Returns TrendReport{thesis, emergingAreas: List<String>,
    gaps: List<String>, analysedAt}.

- 1 Workflow ResearchWorkflow with steps:
  decomposeStep -> [parallel] scoutStep, analyseStep -> joinStep -> synthesiseStep -> guardrailStep -> emitStep.
  decomposeStep calls forAutonomousAgent(LiteratureCoordinator.class, DECOMPOSE).
  scoutStep and analyseStep run in parallel (CompletionStage zip); each governed by
  WorkflowSettings.builder().stepTimeout(ResearchWorkflow::scoutStep, ofSeconds(60)) and
  stepTimeout(ResearchWorkflow::analyseStep, ofSeconds(60)). On either timeout, transition to a
  degradeStep that calls synthesiseStep with whichever side returned, then ends with DigestDegraded.
  synthesiseStep calls forAutonomousAgent(LiteratureCoordinator.class, SYNTHESISE) with the merged
  inputs; give synthesiseStep a 90s stepTimeout. guardrailStep runs the deterministic DOI vetter +
  LLM judge on the synthesised content; on failure, end with DigestBlocked. WorkflowSettings is
  nested inside Workflow — no import.

- 1 EventSourcedEntity ResearchDigestEntity holding state ResearchDigest{digestId, query,
  DigestStatus, Optional<PublicationBundle> publications, Optional<TrendReport> trends,
  Optional<SynthesisedDigest> synthesised, Optional<String> failureReason,
  Optional<Integer> evalScore, Optional<String> evalRationale, Instant createdAt,
  Optional<Instant> finishedAt}. DigestStatus enum: QUEUED, SCANNING, SYNTHESISED,
  DEGRADED, BLOCKED. Events: DigestCreated, PublicationsAttached, TrendsAttached,
  DigestSynthesised, DigestDegraded, DigestBlocked, DigestEvalScored. Commands: createDigest,
  attachPublications, attachTrends, synthesise, degrade, block, recordEval, getDigest.
  emptyState() returns ResearchDigest.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity QueryQueue with command enqueueQuery(query, requestedBy) emitting
  QuerySubmitted{digestId, query, requestedBy, submittedAt}.

- 1 View DigestView with row type ResearchDigestRow (mirrors ResearchDigest minus heavy nested
  payloads; every nullable field is Optional<T>). Table updater consumes ResearchDigestEntity
  events. ONE query getAllDigests SELECT * AS digests FROM digest_view. No WHERE status
  filter (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer QueryRequestConsumer subscribed to QueryQueue events; on QuerySubmitted
  starts a ResearchWorkflow with the digestId as the workflow id.

- 2 TimedActions:
  * QuerySimulator — every 60s, reads next line from
    src/main/resources/sample-events/research-queries.jsonl and calls QueryQueue.enqueueQuery.
  * EvalSampler — every 5 minutes, queries DigestView.getAllDigests, picks the oldest
    SYNTHESISED digest without an evalScore, runs a 1–5 rubric judge over the synthesised
    content, then calls ResearchDigestEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * DigestEndpoint at /api with POST /digests, GET /digests, GET /digests/{id},
    GET /digests/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- AcademicResearchTasks.java declaring four Task<R> constants: DECOMPOSE (ScoutingPlan), SCOUT
  (PublicationBundle), ANALYSE (TrendReport), SYNTHESISE (SynthesisedDigest).
- Domain records ScoutingPlan, Publication, PublicationBundle, TrendReport, SynthesisedDigest.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9686 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/research-queries.jsonl with 8 canned research query lines
  spanning fields such as LLM alignment, CRISPR applications, and quantum error correction.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 citation guardrail
  before-agent-response, E1 eval-event on-decision-eval) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function = academic-
  literature-synthesis, decisions.authority_level = recommend-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/literature-coordinator.md, prompts/paper-scout.md, prompts/domain-analyst.md loaded
  at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Academic Research Team", one-line
  pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills). Browser title exactly: <title>Akka Sample: Academic Research Team</title>. No
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
  src/main/resources/mock-responses/<agent-name>.json (literature-coordinator.json,
  paper-scout.json, domain-analyst.json), picks one entry pseudo-randomly per call,
  and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    literature-coordinator.json — list of either ScoutingPlan or SynthesisedDigest objects.
      4–6 ScoutingPlan entries (scoutingDirective + interpretationQuestion pairs) and
      4–6 SynthesisedDigest entries (each with an 80–150 word summary, a 3–6
      publication bundle, a 3–6 emerging-area trend report, guardrailVerdict = "ok").
    paper-scout.json — 4–6 PublicationBundle entries, each with 3–6 publications
      whose doi values follow realistic formats (e.g., "10.1038/s41586-024-07349-3",
      "10.1126/science.abq3499") and venue values are recognisable journal or
      conference names.
    domain-analyst.json — 4–6 TrendReport entries, each with a one-sentence
      thesis naming a field-level shift, 3–6 short emergingAreas, and 3–6 gaps.
- A MockModelProvider.seedFor(digestId) helper makes the selection
  deterministic per digest id so the same digest produces the same output
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s workers,
  90s synthesis); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion AcademicResearchTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9686 in application.conf.
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
