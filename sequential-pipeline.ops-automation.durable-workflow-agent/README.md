# Akka Sample: Durable Workflow-Backed Agent

An `OpsAgent` walks an ops incident through three task phases — **DIAGNOSE → REMEDIATE → VERIFY** — backed by a durable workflow that guarantees each phase completes even under transient infrastructure failures. A per-task budget cap halts runaway tool loops; a scheduled evaluator monitors workflow durability metrics over a rolling window.

Demonstrates the **sequential-pipeline** coordination pattern in an ops-automation domain, with two governance mechanisms: a `before-tool-call` budget-cap guardrail that halts a task when its tool-call budget is exhausted, and a scheduled `eval-periodic` evaluator that computes MTTR, retry rates, and timeout rates across all completed incidents.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. Every diagnose / remediate / verify tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.ops-automation.durable-workflow-agent  ~/my-projects/durable-workflow-agent
cd ~/my-projects/durable-workflow-agent
```

(Optional) Edit `SPEC.md` to point at a different incident seed set, a different model provider, or a richer set of remediation actions.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **OpsAgent** — one AutonomousAgent declaring three Task constants (`DIAGNOSE_INCIDENT`, `APPLY_REMEDIATION`, `VERIFY_FIX`); the workflow runs them in order, feeding each phase's typed output as the next phase's instruction context.
- **RemediationWorkflow** — runs `diagnoseStep → remediateStep → verifyStep`. Each step calls `runSingleTask` and writes the typed result back onto `IncidentEntity` before the next step starts. Steps carry explicit timeouts; the default recovery policy retries twice before failing over to an error step.
- **IncidentEntity** — an EventSourcedEntity holding the per-incident lifecycle (`DiagnosisCompleted`, `RemediationCompleted`, `VerificationCompleted`, `BudgetExhausted`, `IncidentFailed`).
- **DiagnoseTools / RemediateTools / VerifyTools** — three function-tool classes registered on the agent, one per phase. Reads from `src/main/resources/sample-data/incidents/*.json` for deterministic offline output.
- **BudgetGuardrail** — the `before-tool-call` budget-cap hook. Tracks per-task tool-call count against a configured cap; rejects the call and records a `BudgetExhausted` event when the cap is reached.
- **DurabilityEvaluator** — a TimerService-scheduled evaluator. Runs every hour; queries `IncidentView` for incidents resolved in the trailing window and computes completion rate, retry rate, MTTR, and timeout rate. Writes results to `DurabilityReportEntity`.
- **IncidentView + RemediationEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded incident set under `src/main/resources/sample-events/incidents.jsonl` to fit your environment.
- `SPEC.md §4` and `prompts/ops-agent.md` — narrow the agent's role (e.g., constrain it to Kubernetes events, to cloud-provider alerts, to network anomalies) by tightening the system prompt.
- `SPEC.md §5` — extend the typed outputs (`DiagnosisReport`, `RemediationPlan`, `VerificationResult`) with environment-specific fields. The budget-cap guardrail does not need editing — it tracks call counts, not field shapes.
- `eval-matrix.yaml` — adjust the per-task budget cap in the H1 implementation paragraph; adjust the scheduled-eval window in the E1 implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits an incident → `DIAGNOSE` runs → `REMEDIATE` runs → `VERIFY` runs → a typed `VerificationResult` lands in the UI within ~60 s. Every transition is visible in real time.
2. A task's tool-call count hits the budget cap (forced via the mock LLM) → `BudgetGuardrail` halts the call → `BudgetExhausted` event recorded → workflow retries the step → incident either resolves or lands in `FAILED` after the retry budget is exhausted.
3. The scheduled `DurabilityEvaluator` fires (simulated via a test-endpoint trigger), reads incidents resolved in the last hour, and writes a `DurabilityReport` whose metrics appear in the App UI Overview panel.
4. Each task receives only its own typed inputs; the DIAGNOSE task does not see the remediation instructions, and the VERIFY task does not see raw log fetches — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
