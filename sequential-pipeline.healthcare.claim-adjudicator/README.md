# Akka Sample: Claim Adjudication Agent

A single `AdjudicationAgent` walks an insurance claim through three task phases — **VALIDATE → EVALUATE → ADJUDICATE** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a small set of phase-specific tools. A claims processor submits a claim and receives a structured `AdjudicationDecision`.

Demonstrates the **sequential-pipeline** coordination pattern wired with three governance mechanisms: a `before-tool-call` guardrail that sanitises PHI before any tool processes it, a human-in-the-loop hold for denial decisions, and an `on-decision-eval` evaluator that scores every adjudication for policy-rule coverage and evidence completeness.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every validate / evaluate / adjudicate tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.healthcare.claim-adjudicator  ~/my-projects/claim-adjudicator
cd ~/my-projects/claim-adjudicator
```

(Optional) Edit `SPEC.md` to point at a different payer policy set, a different model provider, or a richer set of evaluation tools.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **AdjudicationAgent** — one AutonomousAgent declaring three Task constants (`VALIDATE_CLAIM`, `EVALUATE_COVERAGE`, `ADJUDICATE_CLAIM`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **ClaimAdjudicationWorkflow** — runs `validateStep → evaluateStep → adjudicateStep → evalStep`. Each step calls `runSingleTask` and writes the typed result back onto `ClaimEntity` before the next step starts. When adjudication produces a denial, the workflow pauses for a human-review step before finalising.
- **ClaimEntity** — an EventSourcedEntity holding the per-claim lifecycle (`ClaimValidated`, `CoverageEvaluated`, `DecisionReached`, `ReviewApproved`, `EvaluationScored`).
- **ValidateTools / EvaluateTools / AdjudicateTools** — three function-tool classes registered on the agent, one per phase. The `before-tool-call` guardrail enforces that PHI is sanitised before any tool body runs and that each tool is only callable in its own phase.
- **PhiSanitizerGuardrail** — the runtime check that strips or masks PHI fields before they enter a tool call. Runs on every tool invocation regardless of phase.
- **AdjudicationScorer** — deterministic, rule-based on-decision evaluator that runs immediately after `DecisionReached` and emits a 1–5 score for policy-rule coverage.
- **ClaimView + ClaimEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded claim set under `src/main/resources/sample-events/claims.jsonl` to fit your payer's specific procedure codes and policy rules.
- `SPEC.md §4` and `prompts/adjudication-agent.md` — narrow the agent's role (e.g., restrict to dental claims, to durable medical equipment, to out-of-network disputes) by tightening the system prompt and renaming the typed records.
- `SPEC.md §5` — extend the typed outputs (`ValidationResult`, `CoverageEvaluation`, `AdjudicationDecision`) with payer-specific fields such as fee schedules or prior-authorisation references.
- `eval-matrix.yaml` — wire a real PHI-detection sanitiser (replace the deterministic stub with a regex-plus-NER implementation) by editing the `S1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A processor submits a claim → `VALIDATE` runs → `EVALUATE` runs → `ADJUDICATE` runs → a typed `AdjudicationDecision` lands in the UI within ~90 s. Every transition is visible in real time.
2. A tool from a later phase is called out of order (forced via the mock LLM) → the `before-tool-call` guardrail rejects the call → the workflow records the rejection event → the agent retries in-phase → the pipeline completes correctly.
3. A claim routed to denial triggers the human-review hold; the UI shows the claim in `PENDING_REVIEW` until a reviewer clicks **Approve denial** or **Override to approve**.
4. Every `AdjudicationDecision` has an on-decision eval score; decisions where fewer than two policy rules were cited receive a score ≤ 2 and are flagged for secondary review.

## License

Apache 2.0.
