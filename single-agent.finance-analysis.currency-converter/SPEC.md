# SPEC — currency-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** CurrencyAgent.
**One-line pitch:** A user submits a currency pair and an amount; one AI agent reads the current rate snapshot (passed as a task attachment, never as inline prompt text) and returns a typed `ConversionResult` — the converted amount, the exchange rate applied, a confidence note, and a brief market-context observation.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the finance-analysis domain. One `ExchangeRateAgent` (AutonomousAgent) carries the entire conversion decision; the surrounding components prepare the rate context and audit the output. There are no controls in the baseline eval-matrix — this is the reference cell before any governance layer is added.

The blueprint shows the minimal wiring for a single-agent finance lookup: one Consumer fetches and attaches the rate context, one Workflow drives the steps, one EventSourcedEntity tracks the lifecycle, and one AutonomousAgent makes the determination. A deployer can layer controls on top by editing `eval-matrix.yaml` and extending the generated components.

## 3. User-facing flows

The user opens the App UI tab.

1. The user fills in a **From currency** (ISO 4217 code, e.g. `USD`), a **To currency** (e.g. `EUR`), and an **Amount**.
2. (Optional) The user picks a **rate snapshot** from a dropdown — one of three seeded snapshots (spot, morning-fix, end-of-day) — or leaves the default (latest).
3. The user clicks **Convert**. The UI POSTs to `/api/conversions` and receives a `conversionId`.
4. The card appears in the live list in `REQUESTED` state. Within ~1 s, it transitions to `RATE_ATTACHED` — the system has loaded the rate context.
5. Within ~10–30 s the agent finishes. The card transitions to `CONVERTING` then `RESULT_RECORDED`. The result appears: the converted amount, the exchange rate used, a one-sentence confidence note, and a two-sentence market-context observation.
6. Within ~1 s of the result, the `evalStep` finishes. The card shows an **eval score** chip (1–5) indicating rate-data freshness and result coherence.
7. The user can submit another conversion; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ConversionEndpoint` | `HttpEndpoint` | `/api/conversions/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `ConversionEntity`, `ConversionView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ConversionEntity` | `EventSourcedEntity` | Per-conversion lifecycle: requested → rate-attached → converting → result → evaluated. Source of truth. | `ConversionEndpoint`, `RateFetcher`, `ConversionWorkflow` | `ConversionView` |
| `RateFetcher` | `Consumer` | Subscribes to `ConversionRequested` events; loads rate snapshot; calls `ConversionEntity.attachRate`. | `ConversionEntity` events | `ConversionEntity` |
| `ConversionWorkflow` | `Workflow` | One workflow per conversion. Steps: `awaitRateStep` → `convertStep` → `evalStep`. | started by `RateFetcher` once rate event lands | `ExchangeRateAgent`, `ConversionEntity` |
| `ExchangeRateAgent` | `AutonomousAgent` | The one decision-making LLM. Receives conversion parameters in the task definition and the rate snapshot as a task attachment; returns `ConversionResult`. | invoked by `ConversionWorkflow` | returns result |
| `ResultGuardrail` | supporting class | `before-agent-response` hook: validates `ConversionResult` structure before the agent returns. | wired on `ExchangeRateAgent` | agent loop |
| `FreshnessScorer` | supporting class | Deterministic rule-based scorer; no LLM call. Runs inside `evalStep`. | invoked by `ConversionWorkflow` | `ConversionEntity` |
| `ConversionView` | `View` | Read model: one row per conversion for the UI. | `ConversionEntity` events | `ConversionEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ConversionRequest(
    String conversionId,
    String fromCurrency,    // ISO 4217
    String toCurrency,      // ISO 4217
    BigDecimal amount,
    String snapshotLabel,   // "spot" | "morning-fix" | "end-of-day"
    String requestedBy,
    Instant requestedAt
) {}

record RateSnapshot(
    String label,
    String fromCurrency,
    String toCurrency,
    BigDecimal rate,
    Instant capturedAt,
    String source           // e.g. "ecb-fix", "spot-feed", "seed-file"
) {}

record ConversionResult(
    BigDecimal convertedAmount,
    BigDecimal rateApplied,
    String fromCurrency,
    String toCurrency,
    String confidenceNote,  // 1 sentence
    String marketContext,   // 1–2 sentences
    Instant decidedAt
) {}

record FreshnessEval(
    int score,              // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record Conversion(
    String conversionId,
    Optional<ConversionRequest> request,
    Optional<RateSnapshot> rateSnapshot,
    Optional<ConversionResult> result,
    Optional<FreshnessEval> eval,
    ConversionStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ConversionStatus {
    REQUESTED, RATE_ATTACHED, CONVERTING, RESULT_RECORDED, EVALUATED, FAILED
}
```

Events on `ConversionEntity`: `ConversionRequested`, `RateAttached`, `ConversionStarted`, `ResultRecorded`, `FreshnessScored`, `ConversionFailed`.

Every nullable lifecycle field on the `Conversion` record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/conversions` — body `{ fromCurrency, toCurrency, amount, snapshotLabel, requestedBy }` → `{ conversionId }`.
- `GET /api/conversions` — list all conversions, newest-first.
- `GET /api/conversions/{id}` — one conversion.
- `GET /api/conversions/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: CurrencyAgent</title>`.

The App UI tab is a two-column layout: a left column with the conversion form and the live list of submitted conversions (status pill + amount chip + age) and a right pane with the selected conversion's detail — rate snapshot used, converted amount, confidence note, market-context, and freshness eval chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The baseline ships with **zero controls** — the eval-matrix's `controls` list is empty. A deployer adding rate-staleness limits, output format guardrails, or PII checks extends `eval-matrix.yaml` and wires the corresponding components. The generated system wires `ResultGuardrail` (structural validator) and `FreshnessScorer` (deterministic staleness scorer) as baseline plumbing — they are ready to be extended but enforce no blocking policy in the baseline tier.

## 9. Agent prompts

- `ExchangeRateAgent` → `prompts/exchange-rate-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached rate snapshot, compute the converted amount, and return a well-formed `ConversionResult`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a USD→EUR conversion with the seeded spot snapshot; within 30 s a `ConversionResult` appears with the converted amount and a non-empty market-context paragraph.
2. **J2** — The agent's first response on a conversion is intentionally malformed (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration produces a well-formed result; the UI never displays the malformed response.
3. **J3** — A conversion using a rate snapshot captured >24 h ago receives a freshness eval score ≤ 2 with a clear rationale; the card border highlights amber.
4. **J4** — Three currency pairs submitted in succession each return unique `conversionId` values and independent results; the live list shows all three.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named currency-agent demonstrating the single-agent × finance-analysis cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-finance-analysis-currency-converter. Java package io.akka.samples.currencyagent.
Akka 3.6.0. HTTP port 9423.

Components to wire (exactly):

- 1 AutonomousAgent ExchangeRateAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/exchange-rate-agent.md>) and
  .capability(TaskAcceptance.of(CONVERT_CURRENCY).maxIterationsPerTask(3)). The task receives
  conversion parameters (fromCurrency, toCurrency, amount, snapshotLabel) as its instruction
  text and the rate snapshot JSON as a task ATTACHMENT (NOT as inline prompt text —
  Akka's TaskDef.attachment(name, contentBytes) is the canonical call). Output:
  ConversionResult{convertedAmount: BigDecimal, rateApplied: BigDecimal,
  fromCurrency: String, toCurrency: String, confidenceNote: String, marketContext: String,
  decidedAt: Instant}. The agent is configured with a before-agent-response guardrail
  (ResultGuardrail) registered via the agent's guardrail-configuration block. On guardrail
  rejection the agent loop retries within its 3-iteration budget.

- 1 Workflow ConversionWorkflow per conversionId with three steps:
  * awaitRateStep — polls ConversionEntity.getConversion every 1s; on
    conversion.rateSnapshot().isPresent() advances to convertStep.
    WorkflowSettings.stepTimeout 15s.
  * convertStep — emits ConversionStarted, then calls componentClient.forAutonomousAgent(
    ExchangeRateAgent.class, "agent-" + conversionId).runSingleTask(
      TaskDef.instructions(formatRequest(conversion.request))
        .attachment("rate-snapshot.json",
          toJson(conversion.rateSnapshot).getBytes())
    ) — returns a taskId, then forTask(taskId).result(CONVERT_CURRENCY) to fetch the result.
    On success calls ConversionEntity.recordResult(result). WorkflowSettings.stepTimeout
    60s with defaultStepRecovery maxRetries(2).failoverTo(ConversionWorkflow::error).
  * evalStep — runs deterministic FreshnessScorer (NOT an LLM call) over the recorded result
    and the attached RateSnapshot: checks that rateSnapshot.capturedAt is within the last 6 h
    (score 5), 6–24 h (score 3), or >24 h (score 1), and applies +1 if confidenceNote is non-
    empty and -1 if marketContext is empty. Emits FreshnessScored{score: 1-5, rationale}.
    WorkflowSettings.stepTimeout 5s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ConversionEntity (one per conversionId). State
  Conversion{conversionId: String, request: Optional<ConversionRequest>,
  rateSnapshot: Optional<RateSnapshot>, result: Optional<ConversionResult>,
  eval: Optional<FreshnessEval>, status: ConversionStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. ConversionStatus enum: REQUESTED, RATE_ATTACHED,
  CONVERTING, RESULT_RECORDED, EVALUATED, FAILED. Events: ConversionRequested{request},
  RateAttached{rateSnapshot}, ConversionStarted{}, ResultRecorded{result},
  FreshnessScored{eval}, ConversionFailed{reason}. Commands: submit, attachRate,
  markConverting, recordResult, recordEvaluation, fail, getConversion. emptyState() returns
  Conversion.initial("") with no commandContext() reference (Lesson 3). Every Optional<T>
  field uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer RateFetcher subscribed to ConversionEntity events; on ConversionRequested reads
  the matching rate snapshot from src/main/resources/sample-data/rate-snapshots.jsonl
  (filtering by fromCurrency, toCurrency, and snapshotLabel), builds RateSnapshot, then calls
  ConversionEntity.attachRate(rateSnapshot). After attachRate lands, the same Consumer starts
  a ConversionWorkflow with id = "conv-" + conversionId.

- 1 View ConversionView with row type ConversionRow (mirrors Conversion). Table updater
  consumes ConversionEntity events. ONE query getAllConversions: SELECT * AS conversions
  FROM conversion_view. No WHERE status filter — Akka cannot auto-index enum columns
  (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ConversionEndpoint at /api with POST /conversions (body
    {fromCurrency, toCurrency, amount, snapshotLabel, requestedBy};
    mints conversionId; calls ConversionEntity.submit; returns {conversionId}),
    GET /conversions (list from getAllConversions, sorted newest-first),
    GET /conversions/{id} (one row), GET /conversions/sse (Server-Sent Events forwarded
    from the view's stream-updates), and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ConversionTasks.java declaring one Task<R> constant: CONVERT_CURRENCY = Task.name("Convert
  currency").description("Read the attached rate snapshot and return a ConversionResult for
  the given currency pair and amount").resultConformsTo(ConversionResult.class). DO NOT
  skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records ConversionRequest, RateSnapshot, ConversionResult, FreshnessEval,
  Conversion, ConversionStatus.

- ResultGuardrail.java implementing the before-agent-response hook. Reads the candidate
  ConversionResult from the LLM response, asserts that convertedAmount and rateApplied are
  positive BigDecimal values, that fromCurrency and toCurrency are non-blank strings, and
  that decidedAt is not null. On failure returns Guardrail.reject(<structured-error>).

- FreshnessScorer.java — pure deterministic logic (no LLM). Inputs: ConversionResult and
  RateSnapshot. Outputs: FreshnessEval. Scoring rubric documented in Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9423 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The ExchangeRateAgent.definition()
  binds the configured provider via the per-agent override pattern from the akka-context docs.

- src/main/resources/sample-data/rate-snapshots.jsonl with seeded rate snapshots covering:
  USD/EUR, USD/GBP, USD/JPY, EUR/GBP, EUR/JPY, GBP/JPY — each pair with three snapshot
  labels (spot, morning-fix, end-of-day) and timestamps spread across the last 48 h so
  the freshness scorer exercises all three score bands.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with controls: [] (baseline tier — no controls).

- risk-survey.yaml at the project root pre-filled for the finance-analysis domain.

- prompts/exchange-rate-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: CurrencyAgent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  conversion form + live list of conversion cards; right = selected-conversion detail with
  rate snapshot info, converted amount, confidence note, market context, and freshness eval
  chip). Browser title exactly: <title>Akka Sample: CurrencyAgent</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent (see Mock LLM provider block below). Sets
        model-provider = mock.
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(conversionId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    convert-currency.json — 8 ConversionResult entries covering a range of currency pairs
      and amounts. Each entry has a positive convertedAmount, a positive rateApplied, non-
      empty confidenceNote and marketContext, and a valid decidedAt. Plus 2 deliberately
      MALFORMED entries (one with a null convertedAmount; one with a blank fromCurrency) —
      the guardrail blocks both, exercising the retry path. The mock selects a malformed
      entry on the FIRST iteration of every 3rd conversion (modulo seed) so J2 is
      reproducible.
- A MockModelProvider.seedFor(conversionId) helper makes per-conversion selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ExchangeRateAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ConversionTasks.java MUST
  exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (convertStep
  60s, awaitRateStep 15s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Conversion record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: ConversionTasks.java with CONVERT_CURRENCY = Task.name(...).description(...)
  .resultConformsTo(ConversionResult.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9423 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (ExchangeRateAgent).
  The freshness eval is rule-based (FreshnessScorer.java) and does NOT make an LLM call —
  keeping the pattern's "one agent" promise honest.
- The rate snapshot is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated convertStep uses TaskDef.attachment(...) and not
  string interpolation into the instruction text.
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
