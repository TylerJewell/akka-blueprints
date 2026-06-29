# Akka Sample: Hierarchical Sub-Agent Delegation

A task orchestrator decomposes a software development task, validates each specialist's input schema before invocation, delegates design work and implementation outlining to two isolated-context specialist sub-agents running in parallel, then synthesises their contributions into one structured solution summary. Demonstrates the **delegation-supervisor-workers** coordination pattern in the dev-code domain.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the inbound task queue and the specialist tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.dev-code.hierarchical-subagents  ~/my-projects/hierarchical-subagents
cd ~/my-projects/hierarchical-subagents
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or output schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TaskOrchestrator** — AutonomousAgent that decomposes a coding task into specialist queries and synthesises the returned contributions.
- **ArchitectSpecialist** — AutonomousAgent that produces a design proposal: approach, component list, and tradeoffs.
- **ImplementationSpecialist** — AutonomousAgent that produces an implementation outline: pseudo-code sketch and risk list.
- **DelegationWorkflow** — Workflow that validates sub-agent inputs before each invocation, fans work out to the two specialists in parallel, then calls TaskOrchestrator for synthesis.
- **TaskDelegationEntity** — EventSourcedEntity holding the task's full lifecycle.
- **TaskQueue** — EventSourcedEntity logging submitted tasks for replay and audit.
- **DelegationView** — projection the UI streams via SSE.
- **TaskQueueConsumer** — Consumer that starts one workflow per submitted task.
- **TaskSimulator** + **ContributionEvaluator** — TimedActions for background load and contribution scoring.
- **TaskEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample tasks the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `DelegationTask` record fields (e.g., add `priority`).
- `prompts/architect-specialist.md` — narrow the specialist to a single technology stack.
- `eval-matrix.yaml` — add a second `before-agent-invocation` guardrail if you integrate a real code-execution sandbox.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a coding task → task enters `SCOPING`, then `DELEGATING`, then `SYNTHESISED`.
2. Worker fails-fast → if either specialist times out, the task enters `DEGRADED` with whichever partial output exists.
3. Schema validation rejects a malformed orchestrator plan → task enters `REJECTED` before any specialist is invoked.
4. ContributionEvaluator scores each specialist's contribution; scores appear on the App UI.

## License

Apache 2.0.
