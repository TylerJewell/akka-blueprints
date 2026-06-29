# Akka Sample: Spec to Implementation PR

A single `ImplementationAgent` walks a product spec through three task phases — **PARSE → PLAN → DRAFT** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a small set of phase-specific tools. The user submits a spec document and receives a draft pull request grounded in the company codebase.

Demonstrates the **sequential-pipeline** coordination pattern with three governance mechanisms: a `before-tool-call` guardrail that blocks repo-write tools when their phase's preconditions are not yet met; a human-in-the-loop (HITL) hold that requires a reviewer to approve the draft PR before it is pushed; and a CI gate that must pass before merge is permitted.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every parse / plan / draft tool is implemented in-process inside the same Akka service. The repo-write and CI-check tools read from and write to an in-process mock repository so no external Git host or CI platform is required.

## Generate the system

```sh
cp -r ./sequential-pipeline.dev-code.spec-to-pr  ~/my-projects/spec-to-pr
cd ~/my-projects/spec-to-pr
```

(Optional) Edit `SPEC.md` to point at a different set of seeded specs, a different model provider, or a richer set of planning tools.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ImplementationAgent** — one AutonomousAgent declaring three Task constants (`PARSE_SPEC`, `PLAN_CHANGES`, `DRAFT_PR`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **SpecPipelineWorkflow** — runs `parseStep → planStep → draftStep → reviewHoldStep → ciCheckStep`. Each step calls `runSingleTask`, writes the typed result back onto `SpecRunEntity`, and either advances or waits for a human signal.
- **SpecRunEntity** — an EventSourcedEntity holding the per-run lifecycle (`SpecParsed`, `PlanProduced`, `PrDrafted`, `ReviewApproved`, `ReviewRejected`, `CiPassed`, `CiFailed`, `RunFailed`).
- **ParseTools / PlanTools / DraftTools** — three function-tool classes registered on the agent, one per phase. The `before-tool-call` guardrail enforces that each tool is only callable in its own phase.
- **WriteGuardrail** — the runtime check backing the repo-write safety contract. Any tool that touches the mock repository is rejected if the entity's current phase precondition has not been satisfied.
- **ReviewHold** — the HITL component. After `PrDrafted`, the workflow pauses and waits for an explicit `POST /api/runs/{id}/approve` or `POST /api/runs/{id}/reject` call from a human reviewer.
- **CiScorer** — deterministic, rule-based CI simulator that runs after `ReviewApproved` and emits a pass/fail verdict with a list of failing checks.
- **SpecRunView + SpecEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded spec set under `src/main/resources/sample-events/specs.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/implementation-agent.md` — narrow the agent's role (e.g., constrain it to a particular language or framework, or restrict it to only modifying test files) by tightening the system prompt.
- `SPEC.md §5` — extend the typed outputs (`ParsedSpec`, `ChangePlan`, `DraftPr`) with codebase-specific fields. The phase-gating guardrail does not need editing — it checks recorded-phase preconditions, not field shapes.
- `eval-matrix.yaml` — wire a real CI runner (replace the deterministic stub with an actual `mvn test` call) by editing the `H1` and `C1` implementation paragraphs.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a spec → `PARSE` runs → `PLAN` runs → `DRAFT` runs → a typed `DraftPr` lands in the UI within ~60 s. Every transition is visible in real time.
2. A tool from a later phase is called out of order (forced via the mock LLM) → the `before-tool-call` guardrail rejects the call → the workflow records the rejection event → the agent retries in-phase → the pipeline completes correctly.
3. A drafted PR waits in `AWAITING_REVIEW`; no merge-eligible state is reached until a human explicitly approves it via `POST /api/runs/{id}/approve`.
4. After approval, the CI scorer runs; a PR with a synthetic failing test is flagged `CI_FAILED` and never transitions to `MERGE_READY`.

## License

Apache 2.0.
