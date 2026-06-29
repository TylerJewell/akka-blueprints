# Akka Sample: Human-in-the-Loop Agent

An agent proposes an action, the workflow pauses at an explicit human approval gate, and a second agent executes the action only after a person approves it.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider during generation (no key needed).
- Host software: None. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./human-in-loop-gate.general.hitl  ~/my-projects/hitl-agent
cd ~/my-projects/hitl-agent
```

(Optional) Edit `SPEC.md` to change the system name or model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- `ProposalAgent` (AutonomousAgent) — analyses a request and returns a typed `ActionProposal{summary, rationale, riskLevel}`.
- `ExecutionAgent` (AutonomousAgent) — executes an approved action and returns a typed `ActionResult{outcome, completedAt}`.
- `ActionWorkflow` (Workflow) — a 3-task graph: propose → await approval → execute.
- `ActionEntity` (EventSourcedEntity) — the action lifecycle and its events.
- `ActionsView` (View) — a read model the UI queries and streams over SSE.
- `ActionEndpoint` + `AppEndpoint` (HttpEndpoints) — the REST/SSE surface and the static UI.

## Customise before generating

- `SPEC.md` Section 1 — the system name.
- `SPEC.md` Section 11 — the model provider and HTTP port.
- `prompts/proposal-agent.md` — the proposal instructions and risk-assessment criteria.

## What gets validated

- Submitting a request produces a proposal that appears in `PROPOSED` with non-empty `summary` and `rationale`.
- Approving a `PROPOSED` action drives it to `EXECUTED` with a non-null `outcome`.
- Rejecting a `PROPOSED` action drives it to terminal `REJECTED` with the reason shown.
- The execute step never runs unless the action is `APPROVED`.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
