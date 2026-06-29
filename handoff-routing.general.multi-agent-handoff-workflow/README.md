# Akka Sample: Multi-Agent Handoff Workflow

A `RouterAgent` examines an incoming task request, classifies it, and hands the same `EXECUTE` task off to a `DataAnalyst`, `ContentWriter`, or `CodeReviewer` specialist that owns the work end-to-end. Demonstrates the **handoff-routing** coordination pattern wired with three governance mechanisms: a before-agent-invocation guardrail on every agent activation, a before-tool-call guardrail on specialist tool use, and an on-decision eval that scores every routing call.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host-software requirement: **None** — this blueprint runs out of the box. The inbound task stream and the outbound result surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./handoff-routing.general.multi-agent-handoff-workflow  ~/my-projects/multi-agent-handoff
cd ~/my-projects/multi-agent-handoff
```

(Optional) Edit `SPEC.md` to add a new specialist domain (add `RESEARCH`, `LEGAL`, etc.) and the matching specialist agent.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TaskSimulator** — TimedAction firing every 30 s that drips canned task requests from a JSONL file into `TaskQueue`.
- **TaskQueue** — EventSourcedEntity append-only log of every inbound task request (audit before any LLM sees it).
- **AdmissionGuardrail** — Consumer that checks each task request against admission criteria before any agent is activated.
- **RouterAgent** — typed Agent that classifies the admitted request into `DATA_ANALYSIS`, `CONTENT_WRITING`, `CODE_REVIEW`, or `UNROUTABLE`.
- **DataAnalyst** — AutonomousAgent that owns the `EXECUTE` task for data-analysis work items.
- **ContentWriter** — AutonomousAgent that owns the `EXECUTE` task for content-writing work items.
- **CodeReviewer** — AutonomousAgent that owns the `EXECUTE` task for code-review work items.
- **RoutingJudge** — typed Agent used by `RoutingEvalScorer` to grade every routing decision against a 1–5 rubric.
- **HandoffWorkflow** — Workflow per task request: admit → route → branch → execute → validate → publish.
- **TaskEntity** — EventSourcedEntity holding each task's lifecycle.
- **TaskView + WorkflowEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **RoutingEvalScorer** — Consumer that listens for `RoutingDecided` events and writes an inline eval score.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the routing taxonomy (add `RESEARCH`, `TRANSLATION`, etc.) and add the matching specialist agent.
- `SPEC.md §5` — extend the `Task` record with deployer-specific fields (`priority`, `deadline`, `projectId`).
- `prompts/router-agent.md` — narrow the classifier rules (confidence thresholds, domain-specific keywords).
- `prompts/data-analyst.md` / `prompts/content-writer.md` / `prompts/code-reviewer.md` — encode output format requirements, length limits, output constraints.
- `eval-matrix.yaml` — swap the in-process admission guard for a real policy engine or OPA sidecar.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Simulator drips a data-analysis task → it is admitted, routed to `DATA_ANALYSIS`, handed off to `DataAnalyst`, and a result is published.
2. Simulator drips a content-writing task → routed to `CONTENT_WRITING`, resolved by `ContentWriter`, published.
3. An unroutable task short-circuits to `REJECTED` without any specialist invocation.
4. A task request that fails admission (policy-violating content or missing required fields) is blocked at the `AdmissionGuardrail` step; no agent is activated.
5. A specialist tool call that fails the before-tool-call guardrail is blocked; the task lands in `TOOL_BLOCKED` for operator review.
6. The eval score (1–5) and rationale appear on every routed task within a few seconds of the routing decision.

## License

Apache 2.0.
