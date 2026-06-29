# Akka Sample: Research Agent

A Research Planner designs a search strategy from a natural-language query, dispatches each step to a Web Searcher or Document Searcher, evaluates the results for citation accuracy, and produces a synthesized report. Demonstrates the **planner-executor** coordination pattern with an embedded eval-event governance control.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** Both search surfaces — web and document — are simulated inside the same Akka service using seeded fixtures.

## Generate the system

```sh
cp -r ./planner-executor.research-intel.research-agent  ~/my-projects/research-agent
cd ~/my-projects/research-agent
```

(Optional) Edit `SPEC.md` to change the query prompts the simulator drips, or to narrow the citation policy.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ResearchPlannerAgent** — AutonomousAgent that maintains a search ledger (query context, missing facts, search plan, current dispatch) and a findings ledger (per-step results, citation status, confidence signals). Decides the next search step and replans when a step yields no usable result.
- **WebSearchAgent** — AutonomousAgent that answers web-search queries from seeded fixtures.
- **DocumentSearchAgent** — AutonomousAgent that retrieves excerpts from indexed document fixtures.
- **ResearchWorkflow** — Workflow with a plan → dispatch → execute → eval → record → decide loop, replan branch, and terminal exit states.
- **ResearchJobEntity** — EventSourcedEntity holding the job lifecycle, both ledgers, the citation evaluation record, and the final report.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag.
- **ResearchJobView** — projection used by the UI.
- **ResearchEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the query prompts the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust `ResearchJob` record fields (e.g., add `confidenceScore`).
- `prompts/research-planner.md` — narrow the planner to a single topic domain.
- `eval-matrix.yaml` — tighten the citation-accuracy threshold or add a second control.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a query → planner designs a strategy, dispatches to web and document searchers, produces a synthesized report within ~3 minutes.
2. A step result that contains no usable citations fails the eval check; the planner replans with a revised strategy.
3. Click **Halt new dispatches** mid-job; the in-flight step finishes; no further dispatches occur; job moves to `HALTED`.
4. A document fixture containing a high-confidence claim is cited in the final report; the UI shows the citation evaluation verdict alongside each finding.

## License

Apache 2.0.
