# Akka Sample: Parallelization Workflow

A task dispatcher runs an input through multiple independent voter agents in parallel — either sectioning the work across distinct subtasks or fanning the same input out to several perspectives — then aggregates their votes into a final decision. Demonstrates the **debate-multi-perspective** coordination pattern with an inline voting-aggregation eval.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — every external surface is modelled inside the service with Akka primitives.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./debate-multi-perspective.general.parallel-voting-workflow  ~/my-projects/parallel-voting-workflow
cd ~/my-projects/parallel-voting-workflow
```

(Optional) Edit `SPEC.md` to change the voter roles, the aggregation quorum, or the model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TaskDispatcher** — AutonomousAgent that decomposes an input into per-voter task briefs, then aggregates the returned votes into a final decision.
- **FeasibilityVoter** — AutonomousAgent voting on whether the proposed action is feasible given stated constraints.
- **RiskVoter** — AutonomousAgent voting on whether the proposed action introduces unacceptable risk.
- **AlignmentVoter** — AutonomousAgent voting on whether the proposed action aligns with declared objectives.
- **AggregationJudge** — AutonomousAgent that scores how well the final decision reflects the individual votes.
- **VotingWorkflow** — Workflow that fans the task out to three voters in parallel, then calls the dispatcher for aggregation and the eval judge for consistency scoring.
- **TaskEntity** — EventSourcedEntity holding the full voting lifecycle.
- **TaskView** — projection used by the UI.
- **TaskEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample tasks the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Vote` or `AggregatedDecision` record fields (e.g., add a `confidence` field).
- `prompts/alignment-voter.md` — narrow the alignment axis to a specific objective framework.
- `eval-matrix.yaml` — add a `before-tool-call` guardrail if you wire a real action executor.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a task → voting enters `SUBMITTED`, then `VOTING`, then `DECIDED` with three voter ballots and one aggregated decision; UI reflects each transition via SSE.
2. A voter timeout drives the task to `PARTIAL` with the final decision aggregated from whichever votes returned.
3. The eval sampler scores per-vote alignment with the aggregated decision and surfaces the score on the App UI.
4. A quorum shortfall — fewer than two votes agree — drives the task to `INCONCLUSIVE`.
5. The AggregationJudge's consistency score appears on the task card within one eval interval of DECIDED.

## License

Apache 2.0.
