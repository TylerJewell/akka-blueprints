# Akka Sample: Human-in-the-Loop Approval Agent

An operations-automation agent proposes a consequential action — such as scaling a resource, triggering a runbook, or modifying a configuration — and the system pauses until a human operator explicitly approves or denies the proposal. Only on approval does the action agent execute the operation.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider during generation (no key needed).
- Host software: None. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./human-in-loop-gate.ops-automation.hitl-approval-agent  ~/my-projects/hitl-approval-agent
cd ~/my-projects/hitl-approval-agent
```

(Optional) Edit `SPEC.md` to change the system name or model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- `ProposalAgent` (AutonomousAgent) — analyzes an operation request and returns a typed `ActionProposal{actionType, target, rationale, estimatedImpact}`.
- `ExecutionAgent` (AutonomousAgent) — executes an approved action and returns a typed `ActionResult{outcome, completedAt, details}`.
- `ApprovalWorkflow` (Workflow) — a 3-task graph: propose → await approval → execute.
- `ActionEntity` (EventSourcedEntity) — the action lifecycle and its events.
- `ActionsView` (View) — a read model the UI queries and streams over SSE.
- `ApprovalEndpoint` + `AppEndpoint` (HttpEndpoints) — the REST/SSE surface and the static UI.

## Customise before generating

- `SPEC.md` Section 1 — the system name.
- `SPEC.md` Section 11 — the model provider and HTTP port.
- `prompts/proposal-agent.md` — the action analysis and proposal instructions.

## What gets validated

- Submitting an operation request produces a proposal in `PROPOSED` with non-empty rationale.
- Approving a `PROPOSED` action drives it to `EXECUTED` with a non-null outcome within ~30 s.
- Denying a `PROPOSED` action drives it to terminal `DENIED` and the reason shows.
- The execution step never runs unless the action is `APPROVED`.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
