# Akka Sample: Financial Advisor Pipeline

A single `FinancialAdvisorAgent` walks a user's financial query through four task phases — **RESEARCH → STRATEGIZE → EXECUTE → ASSESS** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a scoped set of phase-specific tools. The user submits a financial question and receives a structured `FinancialReport` with actionable guidance and a risk assessment.

Demonstrates the **sequential-pipeline** coordination pattern with two governance mechanisms: a `before-agent-response` guardrail that prepends regulated finance-advice disclaimers to every outbound response, and a `sector` sanitizer that strips or redacts non-compliant content patterns before the response reaches the client.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every research / strategy / execution / risk tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.finance-analysis.financial-advisor-pipeline  ~/my-projects/financial-advisor-pipeline
cd ~/my-projects/financial-advisor-pipeline
```

(Optional) Edit `SPEC.md` to adjust the seeded query set, target a different model provider, or extend the tool set with real market-data APIs.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **FinancialAdvisorAgent** — one AutonomousAgent declaring four Task constants (`RESEARCH_MARKET`, `DEFINE_STRATEGY`, `PLAN_EXECUTION`, `ASSESS_RISK`); the workflow runs them in order, feeding each typed output forward as the next task's instruction context.
- **AdvisorPipelineWorkflow** — runs `researchStep → strategyStep → executionStep → riskStep`. Each step calls `runSingleTask` and writes the typed result back onto `AdvisoryEntity` before the next step starts.
- **AdvisoryEntity** — an EventSourcedEntity holding the per-advisory lifecycle (`MarketResearched`, `StrategyDefined`, `ExecutionPlanned`, `RiskAssessed`).
- **ResearchTools / StrategyTools / ExecutionTools / RiskTools** — four function-tool classes registered on the agent, one per phase. The `before-agent-response` guardrail enforces that regulated disclaimers appear on every outbound LLM response.
- **DisclaimerGuardrail** — the runtime check that injects the required finance-advice disclaimer block before the agent response reaches the workflow or the client. Ensures no advisory output leaves the agent without the regulatory notice.
- **SectorSanitizer** — strips content patterns that violate financial sector compliance rules (e.g. guaranteed-return language, unlicensed securities recommendations) before the response is written to the entity.
- **ComplianceScorer** — deterministic, rule-based on-decision evaluator that runs immediately after `RiskAssessed` and emits a 1–5 score on disclosure completeness and risk coverage.
- **AdvisoryView + AdvisoryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded query set under `src/main/resources/sample-events/queries.jsonl` to fit your demo audience (retirement planning, portfolio rebalancing, tax-loss harvesting, etc.).
- `SPEC.md §4` and `prompts/financial-advisor-agent.md` — narrow the agent's scope (e.g., restrict it to equity-only analysis, or to a single regulatory jurisdiction) by tightening the system prompt and the phase-tool lists.
- `SPEC.md §5` — extend the typed outputs (`MarketSnapshot`, `Strategy`, `ExecutionPlan`, `RiskProfile`) with instrument-specific fields. The disclaimer guardrail does not need editing — it fires on every outbound response regardless of field shape.
- `eval-matrix.yaml` — wire a real sector-rule engine (replace the deterministic sanitizer stub with a call to a compliance rule library) by editing the `S1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a financial query → `RESEARCH` runs → `STRATEGIZE` runs → `EXECUTE` runs → `ASSESS` runs → a typed `FinancialReport` lands in the UI within ~90 s. Every phase transition is visible in real time.
2. The disclaimer guardrail fires on every outbound agent response; the UI's disclaimer-log strip shows each injected notice header with its timestamp.
3. The sector sanitizer detects a prohibited phrase (guaranteed returns) in a mock-LLM response and redacts it before writing the result onto the entity; the sanitizer event is logged and visible.
4. Every `FinancialReport` emitted has an on-decision compliance score visible on the same UI card; reports missing a risk rating or lacking diversification guidance receive a score ≤ 2 and are flagged.

## License

Apache 2.0.
