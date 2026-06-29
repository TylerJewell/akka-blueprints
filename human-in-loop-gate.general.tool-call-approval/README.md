# Akka Sample: Human-in-the-Loop Tool Approval

An agent is given a task that requires calling a tool. The workflow pauses before that tool executes so a human can approve, edit the parameters, or reject the call entirely. The tool runs only after explicit human authorization.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider during generation (no key needed).
- Host software: None. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./human-in-loop-gate.general.tool-call-approval  ~/my-projects/tool-approval
cd ~/my-projects/tool-approval
```

(Optional) Edit `SPEC.md` to change the system name or model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- `PlannerAgent` (AutonomousAgent) — receives a goal, selects a tool, and returns a typed `ToolCallPlan{toolName, parameters, rationale}`.
- `ExecutorAgent` (AutonomousAgent) — executes an approved tool call and returns a typed `ToolCallResult{output, executedAt}`.
- `ApprovalWorkflow` (Workflow) — a 3-task graph: plan → await human decision → execute.
- `ToolRequestEntity` (EventSourcedEntity) — the tool-request lifecycle and its events.
- `ToolRequestsView` (View) — a read model the UI queries and streams over SSE.
- `ApprovalEndpoint` + `AppEndpoint` (HttpEndpoints) — the REST/SSE surface and the static UI.

## Customise before generating

- `SPEC.md` Section 1 — the system name.
- `SPEC.md` Section 11 — the model provider and HTTP port.
- `prompts/planner-agent.md` — the tool selection and reasoning instructions.

## What gets validated

- Submitting a goal produces a `ToolCallPlan` that appears in `PENDING_APPROVAL` with a non-empty tool name and parameters.
- Approving a `PENDING_APPROVAL` request drives it to `EXECUTED` with a non-null output.
- Rejecting a `PENDING_APPROVAL` request drives it to terminal `REJECTED` with the reason shown.
- The execute step never runs unless the request is `APPROVED`.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
