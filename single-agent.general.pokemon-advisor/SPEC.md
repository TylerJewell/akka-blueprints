# SPEC — pokemon-advisor

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** PokemonAdvisor.
**One-line pitch:** A trainer submits their current roster and a target battle format; one AI agent reads the roster (passed as a task attachment, never as inline prompt text) and returns a structured team recommendation — a scored composition with a slot-by-slot rationale and a list of coverage gaps.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in a general educational domain. One `TeamAdvisorAgent` (AutonomousAgent) carries the entire decision; the surrounding components only prepare its input and audit its output. Two governance mechanisms are wired around the agent:

- A **before-agent-response guardrail** validates the agent's recommendation on every turn: well-formed JSON, every slot references a valid species name, every coverage type is in the allowed set, and the recommendation's slots count matches the format's team size. A malformed recommendation triggers a retry inside the same task.
- An **on-decision-eval** runs immediately after each `RecommendationRecorded` event, scoring the recommendation on coverage completeness (does the team cover the six primary damage types? are weaknesses called out? is the reasoning slot-by-slot or just generic?). The eval is deterministic — no LLM call — keeping the single-agent invariant honest.

The blueprint shows that even a toy domain benefits from output validation and quality scoring around the single decision-making model call.

## 3. User-facing flows

The user opens the App UI tab.

1. The user fills in a **Roster** — up to 6 Pokemon species names — and picks a **Battle format** from a dropdown (VGC, Singles, Double-Battle, Casual).
2. The user enters a **Trainer name** and clicks **Get advice**. The UI POSTs to `/api/advisories` and receives an `advisoryId`.
3. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `VALIDATED` — the validator confirmed no duplicate species and the slot count is within the format's limit.
4. Within ~10–30 s, the workflow's `adviseStep` completes. The card transitions to `ADVISING` then `RECOMMENDATION_RECORDED`. The recommendation appears: a team composition table (slot, species, role, key moves) and a coverage gap list.
5. Within ~1 s of the recommendation, the `scoreStep` finishes. The card shows a **coverage score** chip (1–5) plus a one-line rationale describing whether the team's type coverage is balanced.
6. The trainer can submit another roster; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AdvisoryEndpoint` | `HttpEndpoint` | `/api/advisories/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `RosterEntity`, `AdvisoryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `RosterEntity` | `EventSourcedEntity` | Per-advisory lifecycle: submitted → validated → advising → recommendation → scored. Source of truth. | `AdvisoryEndpoint`, `RosterValidator`, `AdvisoryWorkflow` | `AdvisoryView` |
| `RosterValidator` | `Consumer` | Subscribes to `RosterSubmitted` events; checks legality (no duplicates, correct slot count, no banned species for format); calls `RosterEntity.attachValidated`. | `RosterEntity` events | `RosterEntity` |
| `AdvisoryWorkflow` | `Workflow` | One workflow per advisory. Steps: `awaitValidatedStep` → `adviseStep` → `scoreStep`. | started by `RosterValidator` once validated event lands | `TeamAdvisorAgent`, `RosterEntity` |
| `TeamAdvisorAgent` | `AutonomousAgent` | The one decision-making LLM. Receives battle format constraints in the task definition and the validated roster as a task attachment; returns `TeamRecommendation`. | invoked by `AdvisoryWorkflow` | returns recommendation |
| `AdvisoryView` | `View` | Read model: one row per advisory for the UI. | `RosterEntity` events | `AdvisoryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record RosterSlot(int slotNumber, String species, String nickname) {}

record RosterSubmission(
    String advisoryId,
    String trainerName,
    List<RosterSlot> slots,
    BattleFormat format,
    Instant submittedAt
) {}

record ValidatedRoster(
    List<RosterSlot> slots,
    BattleFormat format,
    List<String> warningsFound
) {}

record SlotRecommendation(
    int slotNumber,
    String species,
    Role role,
    List<String> suggestedMoves,
    String rationale
) {}
enum Role { PHYSICAL_SWEEPER, SPECIAL_SWEEPER, WALL, SUPPORT, LEAD, UTILITY }

record CoverageGap(String type, GapSeverity severity, String suggestion) {}
enum GapSeverity { MINOR, MODERATE, SIGNIFICANT }

record TeamRecommendation(
    Verdict verdict,
    String summary,
    List<SlotRecommendation> slots,
    List<CoverageGap> gaps,
    Instant decidedAt
) {}
enum Verdict { STRONG, VIABLE, NEEDS_ADJUSTMENT }

record CoverageScore(
    int score,            // 1..5
    String rationale,
    Instant scoredAt
) {}

record Advisory(
    String advisoryId,
    Optional<RosterSubmission> submission,
    Optional<ValidatedRoster> validated,
    Optional<TeamRecommendation> recommendation,
    Optional<CoverageScore> coverageScore,
    AdvisoryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum BattleFormat { VGC, SINGLES, DOUBLE_BATTLE, CASUAL }

enum AdvisoryStatus {
    SUBMITTED, VALIDATED, ADVISING, RECOMMENDATION_RECORDED, SCORED, FAILED
}
```

Events on `RosterEntity`: `RosterSubmitted`, `RosterValidated`, `AdvisingStarted`, `RecommendationRecorded`, `CoverageScored`, `AdvisoryFailed`.

Every nullable lifecycle field on the `Advisory` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/advisories` — body `{ trainerName, slots: [RosterSlot], format: BattleFormat }` → `{ advisoryId }`.
- `GET /api/advisories` — list all advisories, newest-first.
- `GET /api/advisories/{id}` — one advisory.
- `GET /api/advisories/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Pokemon Team Advisor</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted advisories (status pill + verdict badge + age) and a right pane with the selected advisory's detail — submitted roster slots, validation warnings, recommendation summary, slot-by-slot table, coverage gaps list, and coverage score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. This baseline has zero regulation anchors (the domain is general/educational). The generated system wires:

- **G1 — before-agent-response guardrail**: runs on every turn of `TeamAdvisorAgent`. Asserts the candidate response is well-formed `TeamRecommendation` JSON, every `slots[].species` is a non-empty string, every `slots[].role` is in `{PHYSICAL_SWEEPER, SPECIAL_SWEEPER, WALL, SUPPORT, LEAD, UTILITY}`, every `gaps[].severity` is in `{MINOR, MODERATE, SIGNIFICANT}`, and the `slots` count matches the format's expected team size. On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.
- **E1 — on-decision eval** (`eval-event`, `on-decision-eval`): runs immediately after `RecommendationRecorded` lands, as `scoreStep` inside the workflow. A deterministic rule-based `CoverageScorer` (no LLM call) checks that the team covers at least four of the six primary damage types (Fire, Water, Grass, Electric, Fighting, Psychic), that every slot recommendation contains at least one suggested move, that every gap has a non-empty suggestion, and that severity distributions are realistic. Emits `CoverageScored` with a 1–5 score and a one-line rationale.

## 9. Agent prompts

- `TeamAdvisorAgent` → `prompts/team-advisor.md`. The single decision-making LLM. System prompt instructs it to read the attached roster, apply the format's constraints, and return a `TeamRecommendation` with a slot-by-slot rationale and a list of coverage gaps.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Trainer submits a 6-slot roster for VGC format; within 30 s the recommendation appears with one `SlotRecommendation` per slot and a coverage score chip.
2. **J2** — The agent's first response on an advisory is intentionally malformed (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration produces a well-formed recommendation; the UI never displays the malformed response.
3. **J3** — A recommendation whose `gaps` list is empty and whose team covers only two damage types receives a coverage score of 1 with a clear rationale; the UI flags the card.
4. **J4** — A roster containing a duplicate species (e.g., two Charizard) is submitted; `RosterValidator` emits `AdvisoryFailed` with a `DUPLICATE_SPECIES` reason; the card reaches `FAILED` state; the agent is never called.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named pokemon-advisor demonstrating the single-agent × general cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-pokemon-advisor. Java package io.akka.samples.pokemonteamadvisor.
Akka 3.6.0. HTTP port 9830.

Components to wire (exactly):

- 1 AutonomousAgent TeamAdvisorAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/team-advisor.md>) and
  .capability(TaskAcceptance.of(ADVISE_TEAM).maxIterationsPerTask(3)). The task receives
  battle format constraints as its instruction text and the validated roster as a task
  ATTACHMENT (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is
  the canonical call). Output: TeamRecommendation{verdict: Verdict
  (STRONG/VIABLE/NEEDS_ADJUSTMENT), summary: String, slots: List<SlotRecommendation>,
  gaps: List<CoverageGap>, decidedAt: Instant}. The agent is configured with a
  before-agent-response guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries the response
  within its 3-iteration budget.

- 1 Workflow AdvisoryWorkflow per advisoryId with three steps:
  * awaitValidatedStep — polls RosterEntity.getAdvisory every 1s; on advisory.validated().isPresent()
    advances to adviseStep. WorkflowSettings.stepTimeout 15s (validator is in-process and fast).
  * adviseStep — emits AdvisingStarted, then calls componentClient.forAutonomousAgent(
    TeamAdvisorAgent.class, "advisor-" + advisoryId).runSingleTask(
      TaskDef.instructions(formatConstraints(advisory.submission.format))
        .attachment("roster.txt", formatRoster(advisory.validated).getBytes())
    ) — returns a taskId, then forTask(taskId).result(ADVISE_TEAM) to fetch the recommendation.
    On success calls RosterEntity.recordRecommendation(recommendation).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(AdvisoryWorkflow::error).
  * scoreStep — runs a deterministic rule-based CoverageScorer (NOT an LLM call) over the
    recorded recommendation: checks type-coverage breadth (at least 4 of 6 primary types
    covered by the team's suggested moves), that every slot has at least one suggested move,
    that every gap has a non-empty suggestion, and that gap severities are not uniformly
    SIGNIFICANT (all-max is suspect). Emits CoverageScored{score: 1-5, rationale: String}.
    WorkflowSettings.stepTimeout 5s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity RosterEntity (one per advisoryId). State Advisory{advisoryId: String,
  submission: Optional<RosterSubmission>, validated: Optional<ValidatedRoster>,
  recommendation: Optional<TeamRecommendation>, coverageScore: Optional<CoverageScore>,
  status: AdvisoryStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  AdvisoryStatus enum: SUBMITTED, VALIDATED, ADVISING, RECOMMENDATION_RECORDED, SCORED,
  FAILED. Events: RosterSubmitted{submission}, RosterValidated{validated},
  AdvisingStarted{}, RecommendationRecorded{recommendation}, CoverageScored{coverageScore},
  AdvisoryFailed{reason}. Commands: submit, attachValidated, markAdvising,
  recordRecommendation, recordCoverageScore, fail, getAdvisory. emptyState() returns
  Advisory.initial("") with no commandContext() reference (Lesson 3). Every Optional<T>
  field uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer RosterValidator subscribed to RosterEntity events; on RosterSubmitted runs
  a legality pipeline: checks for duplicate species (case-insensitive), verifies slot count
  is within the format's allowed range (VGC/DOUBLE_BATTLE: 4–6 slots, SINGLES/CASUAL: 1–6
  slots), collects any warnings (e.g., legendary usage in CASUAL), builds ValidatedRoster
  carrying the validated slots, format, and warnings list. Calls
  RosterEntity.attachValidated(validated). After attachValidated lands, the same Consumer
  starts an AdvisoryWorkflow with id = "advisory-" + advisoryId. On duplicate-species
  detection, calls RosterEntity.fail("DUPLICATE_SPECIES") instead of attachValidated and
  does NOT start the workflow.

- 1 View AdvisoryView with row type AdvisoryRow (mirrors Advisory). Table updater consumes
  RosterEntity events. ONE query getAllAdvisories: SELECT * AS advisories FROM advisory_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * AdvisoryEndpoint at /api with POST /advisories (body
    {trainerName, slots: [{slotNumber, species, nickname}], format: BattleFormat};
    mints advisoryId; calls RosterEntity.submit; returns {advisoryId}), GET /advisories
    (list from getAllAdvisories, sorted newest-first), GET /advisories/{id} (one row), GET
    /advisories/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- AdvisoryTasks.java declaring one Task<R> constant: ADVISE_TEAM = Task.name("Advise team")
  .description("Read the attached roster and produce a TeamRecommendation with slot rationale
  and coverage gaps").resultConformsTo(TeamRecommendation.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records RosterSlot, RosterSubmission, ValidatedRoster, SlotRecommendation, Role,
  CoverageGap, GapSeverity, TeamRecommendation, Verdict, CoverageScore, Advisory,
  BattleFormat, AdvisoryStatus.

- RecommendationGuardrail.java implementing the before-agent-response hook. Reads the
  candidate TeamRecommendation from the LLM response, runs the four checks listed in
  eval-matrix.yaml G1, and either passes the response through or returns
  Guardrail.reject(<structured-error>) to force the agent loop to retry.

- CoverageScorer.java — pure deterministic logic (no LLM). Inputs: TeamRecommendation and
  the ValidatedRoster. Outputs: CoverageScore. Scoring rubric documented in Javadoc on the
  class. Type-coverage map is a static Map<String, List<String>> species→primary-types
  covering the seeded roster species.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9830 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The TeamAdvisorAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/rosters.jsonl with 3 seeded roster examples:
  a 6-slot VGC-style team (Incineroar, Miraidon, Flutter Mane, Urshifu, Rillaboom, Amoonguss),
  a 4-slot casual team (Charizard, Blastoise, Venusaur, Pikachu),
  a 6-slot Singles team (Garchomp, Clefable, Corviknight, Heatran, Toxapex, Kartana).
  Each roster has a matching format and trainerName.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, E1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors — domain is
  general/educational with no regulatory surface.

- risk-survey.yaml at the project root with data.data_classes.pii = false (no personal
  identifiers beyond trainer name — a toy system), decisions.authority_level = inform-only
  (the agent's recommendation is advisory, not enforced), oversight.human_in_loop = true
  (trainer reads the recommendation before team-locking), failure.failure_modes including
  "hallucinated-species", "missing-slot", "role-miscategorisation",
  "coverage-gap-omission"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/team-advisor.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Pokemon Team Advisor", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of advisory cards; right = selected-advisory detail with roster slots, validation
  warnings, recommendation summary, slot table, coverage gaps, and coverage score chip).
  Browser title exactly: <title>Akka Sample: Pokemon Team Advisor</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build
        forwards the value from the Claude session env to the JVM via the MCP tool's
        environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml;
        /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using the
        matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(advisoryId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    advise-team.json — 8 TeamRecommendation entries covering the three Verdict values.
      Each entry has a summary paragraph and a `slots` array with one SlotRecommendation per
      roster slot matching the seeded team (VGC 6-slot / casual 4-slot / Singles 6-slot).
      Each SlotRecommendation has a non-empty species, a valid Role, at least one
      suggestedMoves entry, and a one-sentence rationale. Gaps vary (0–4 items per entry).
      Plus 2 deliberately MALFORMED entries (one with a slot whose role is outside the enum;
      one with a missing slots array) — the guardrail blocks both, exercising the retry path.
      The mock selects a malformed entry on the FIRST iteration of every 3rd advisory
      (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(advisoryId) helper makes per-advisory selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. TeamAdvisorAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion AdvisoryTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (adviseStep
  60s, awaitValidatedStep 15s, scoreStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Advisory row record is Optional<T>. The
  view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: AdvisoryTasks.java with ADVISE_TEAM = Task.name(...).description(...)
  .resultConformsTo(TeamRecommendation.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9830 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words in narrative — shape, minimal, smaller, complex, Akka SDK
  in narrative prose, marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (TeamAdvisorAgent). The
  on-decision eval is rule-based (CoverageScorer.java) and does NOT make an LLM call —
  keeping the pattern's "one agent" promise honest.
- The roster is passed as a Task ATTACHMENT, never inlined into the agent's instructions.
  Verify the generated adviseStep uses TaskDef.attachment(...) and not string interpolation
  into the instruction text.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent returns.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block. Per
  Lesson 25, /akka:specify handles the key during generation.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
