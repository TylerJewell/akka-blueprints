# Akka Sample: LATS Tree Search

A `SearchAgent` expands candidate next-actions at each tree node; a `ReflectorAgent` scores each candidate and provides a structured critique; the workflow uses backpropagated scores to select the best branch and continues expanding until a terminal node is reached or the budget is exhausted. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; the problem input and the tree exploration are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.general.lats-tree-search  ~/my-projects/lats-tree-search
cd ~/my-projects/lats-tree-search
```

(Optional) Edit `SPEC.md` to change the problem domain, the expansion branching factor, the score threshold for acceptance, or the node budget.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **SearchAgent** — AutonomousAgent that expands a tree node by generating candidate next-actions (the "expansion" phase of MCTS).
- **ReflectorAgent** — AutonomousAgent that scores each candidate on a 1–10 scale, providing a structured critique and a backpropagation delta.
- **TreeSearchWorkflow** — Workflow that orchestrates the expand → reflect → select → expand cycle, halts when a terminal node is found or the node budget is exhausted, and surfaces the best path found.
- **SearchTreeEntity** — EventSourcedEntity holding the full tree state: every node, every candidate expansion, every reflection score, and the best-path cursor.
- **ProblemQueue** — EventSourcedEntity that logs each submitted problem for replay and audit.
- **TreeView** — read-side projection that the UI lists and streams via SSE.
- **ProblemConsumer** — Consumer that starts a workflow per inbound problem submission.
- **ProblemSimulator** — TimedAction that drips a sample problem every 90 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-node eval event each cycle (control E1).
- **SearchEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample problems the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `SearchTree` record fields (e.g., raise `nodeBudget`, change the branching factor `expansionWidth`).
- `prompts/search-agent.md` — narrow expansion to a specific problem class (e.g., code debugging, multi-step reasoning).
- `prompts/reflector-agent.md` — change the scoring rubric (e.g., weight novelty over correctness).
- `eval-matrix.yaml` — tighten enforcement on the eval-event (currently non-blocking advisory).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a problem → tree progresses `EXPANDING` → `REFLECTING` → `SOLVED` within the node budget.
2. Force-exhaust budget → tree reaches `BUDGET_EXHAUSTED` after the configured node count; the entity preserves every node and every reflection for audit.
3. A node whose expansion count exceeds the branching-factor ceiling is pruned before the Reflector scores it, so the Reflector never sees over-budget expansions.
4. Each completed reflection cycle emits an `EvalRecorded` event that surfaces in the App UI's per-node timeline.

## License

Apache 2.0.
