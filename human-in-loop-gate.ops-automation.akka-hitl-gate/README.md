# Akka Sample: HITL Ops Automation Gate

An operations agent proposes an automated action, the workflow pauses at a human approval gate, and an execution agent carries out the action only after a person approves it.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider during generation (no key needed).
- Host software: None. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./human-in-loop-gate.ops-automation.akka-hitl-gate  ~/my-projects/akka-hitl-gate
cd ~/my-projects/akka-hitl-gate
```

(Optional) Edit `SPEC.md` to change the system name or model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- `ProposalAgent` (AutonomousAgent) — analyzes an ops task request and returns a typed `ActionProposal{summary, estimatedImpact, actionPayload}`.
- `ExecutionAgent` (AutonomousAgent) — executes an approved action and returns a typed `ActionResult{outcome, executedAt}`.
- `GateWorkflow` (Workflow) — a 3-task graph: propose → await approval → execute.
- `ActionEntity` (EventSourcedEntity) — the action lifecycle and its events.
- `ActionsView` (View) — a read model the UI queries and streams over SSE.
- `GateEndpoint` + `AppEndpoint` (HttpEndpoints) — the REST/SSE surface and the static UI.

## Customise before generating

- `SPEC.md` Section 1 — the system name.
- `SPEC.md` Section 11 — the model provider and HTTP port.
- `prompts/proposal-agent.md` — the analysis and proposal-writing instructions.

## What gets validated

- Submitting an ops task request produces a proposal that appears in `PROPOSED` with non-empty `summary` and `actionPayload`.
- Approving a `PROPOSED` action drives it to `EXECUTED` with a non-null `outcome` within ~30 s.
- Rejecting a `PROPOSED` action drives it to terminal `REJECTED` with the reason shown.
- The execution step never runs unless the action is `APPROVED`.

Full journeys: `reference/user-journeys.md`.

## ASCII architecture

```
Browser / API client
        |
        v
 GateEndpoint (/api/*)
  |    |    |    |
  |    |    |  ActionsView (SSE)
  |    |    |        ^
  |    |  ActionEntity (events)
  |    |    ^    ^
  |  GateWorkflow
  |    |        |
  | ProposalAgent  ExecutionAgent
  |
 AppEndpoint (static UI)
```

## Project layout (generated)

```
src/main/java/io/akka/samples/humanintheloop/
  application/
    ProposalAgent.java
    ExecutionAgent.java
    GateTasks.java
    GateWorkflow.java
    ActionEntity.java
    ActionsView.java
  api/
    GateEndpoint.java
    AppEndpoint.java
  domain/
    Action.java
    ActionStatus.java
    ActionProposal.java
    ApprovalDecision.java
    ActionResult.java
    events/
      ActionProposed.java  ActionApproved.java
      ActionRejected.java  ActionExecuted.java
src/main/resources/
  application.conf
  metadata/
    eval-matrix.yaml  risk-survey.yaml  README.md
  static-resources/
    index.html
```

## API contract (summary)

```
POST /api/action-request               -> { actionId }
POST /api/actions/{actionId}/approve   -> 200 | 404
POST /api/actions/{actionId}/reject    -> 200 | 404
GET  /api/actions                      -> { actions: [...] }
GET  /api/actions/{actionId}           -> Action | 404
GET  /api/actions/sse                  -> SSE stream of Action
GET  /api/metadata/eval-matrix         -> text/yaml
GET  /api/metadata/risk-survey         -> text/yaml
GET  /api/metadata/readme              -> text/markdown
GET  /                                 -> 302 /app/index.html
GET  /app/{*path}                      -> static-resources/{*path}
```

## Five UI tabs

1. **Overview** — sample description, Try it card (`/akka:build`), component list.
2. **Architecture** — mermaid component graph, sequence, state machine, ER diagram.
3. **Risk Survey** — pre-filled answers from `risk-survey.yaml`; deployer placeholders muted.
4. **Eval Matrix** — one row per control with mechanism pill and rationale.
5. **App UI** — submit a task request; live action list via SSE; Approve/Reject on `PROPOSED` actions.

## License

Apache 2.0.
