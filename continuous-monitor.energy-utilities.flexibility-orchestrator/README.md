# Akka Sample: Demand-Flexibility Orchestrator

A continuous background worker polls a grid demand signal feed, evaluates curtailment needs with an AI agent, and dispatches flexibility programs — holding every high-threshold dispatch for operator approval and enforcing a hard curtailment budget cap per program window. Demonstrates the **continuous-monitor** coordination pattern wired with two governance mechanisms (HITL approval gate, halt budget-cap).

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./continuous-monitor.energy-utilities.flexibility-orchestrator  ~/my-projects/flexibility-orchestrator
cd ~/my-projects/flexibility-orchestrator
```

(Optional) Edit `SPEC.md` to point `GridSignalPoller` at a real ISO/utility signal feed or to keep the in-memory simulator.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **GridSignalPoller** — TimedAction firing every 60 s that inserts simulated grid demand readings into `DemandSignalEntity`.
- **DispatchEvaluatorAgent** — Agent (typed) that analyzes each new demand reading and recommends a dispatch action (curtailment MW, pricing delta, urgency level).
- **DispatchWorkflow** — per-request orchestration: evaluate → budget check → (if above threshold) await operator approval → execute or cancel.
- **FlexibilityProgramEntity** — EventSourcedEntity tracking per-program budget, dispatch history, and window status.
- **DispatchEntity** — EventSourcedEntity holding each dispatch request's lifecycle (requested → awaiting_approval → approved/cancelled → executing → completed).
- **PricingConsumer** — Consumer that subscribes to `DispatchApproved` events and calculates real-time pricing adjustments.
- **DispatchView** — read model for the operator dashboard.
- **EvalRunner** — TimedAction running every 30 minutes; samples completed dispatches and scores their accuracy.
- **FlexibilityEndpoint + AppEndpoint** — REST/SSE + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the simulated signal feed for a real ISO or SCADA integration.
- `SPEC.md §5` — add org-specific fields to `DispatchRequest` (`programType`, `assetGroup`, `customerSegment`).
- `prompts/dispatch-evaluator.md` — narrow the evaluation rubric to your grid's specific frequency and voltage thresholds.
- `eval-matrix.yaml` — adjust the approval threshold MW value under the HITL control's implementation block.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A simulated demand signal arrives with ELEVATED severity; the AI recommends curtailment; the dispatch is held for operator approval.
2. Operator approves the dispatch; it transitions to EXECUTING then COMPLETED.
3. A dispatch request that would exceed the program's curtailment budget is halted before approval is even requested.
4. Operator cancels a pending dispatch; the program budget is not debited.
5. EvalRunner scores at least one COMPLETED dispatch within 30 minutes of system start.

## License

Apache 2.0.
