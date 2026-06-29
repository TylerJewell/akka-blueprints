# Akka Sample: Computer Use Agent

A single computer-use agent receives a task description and executes it by operating a desktop or web environment through screen observation, keyboard input, and mouse actions. Every tool call (click, type, screenshot, scroll) passes through a before-tool-call guardrail that blocks destructive or out-of-scope actions. An operator-controlled halt endpoint stops the agent mid-task at any moment. High-impact actions — form submissions, file deletions, purchases — pause for explicit user confirmation before proceeding.

Demonstrates the **single-agent** coordination pattern in the ops-automation domain, wired with three governance mechanisms: a before-tool-call guardrail that evaluates each action before execution, an operator halt that can interrupt the running agent at any point, and a human-in-the-loop confirmation gate that surfaces high-risk actions to the user before they fire.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software required. The agent's tool calls target a sandboxed in-process browser simulator; no real desktop or browser automation library is needed to run the sample.

## Generate the system

```sh
cp -r ./single-agent.ops-automation.computer-use-agent  ~/my-projects/computer-use-agent
cd ~/my-projects/computer-use-agent
```

(Optional) Edit `SPEC.md` to change the seeded task library (e.g., swap the generic form-filling tasks for a SaaS-portal automation workflow or a file-management sequence).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. `SPEC.md`'s Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ComputerUseAgent** — an AutonomousAgent that receives a task description plus a screenshot attachment and returns a `ToolCall` (the next action to execute) or a `TaskOutcome` when done.
- **TaskExecutionWorkflow** — orchestrates validate → execute → confirm (when needed) → observe in a loop per submitted task.
- **TaskEntity** — an EventSourcedEntity holding the per-task lifecycle and the action history.
- **ActionGuardrail** — a before-tool-call guardrail that runs before every action, blocking destructive or out-of-policy operations.
- **ConfirmationGateway** — a Consumer that intercepts high-impact actions and writes a `ConfirmationPending` event; the workflow waits for the user's explicit approval.
- **TaskView + TaskEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded task library for your own (the JSONL file under `src/main/resources/sample-events/seed-tasks.jsonl` after generation).
- `SPEC.md §5` — extend `ToolCall` with environment-specific action types (e.g., add `API_CALL` or `SHELL_COMMAND` action types for a server-automation variant).
- `prompts/computer-use-agent.md` — narrow the agent's permitted action scope (a finance deployer would constrain it to read-only portal navigation; an IT deployer might allow controlled script execution).
- `eval-matrix.yaml` — wire a real sandbox environment (e.g., a Playwright browser or a VNC session) by naming it under the guardrail mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a task; the agent executes it step-by-step; each action is logged; the task completes with a `TaskOutcome` visible in the UI.
2. The agent attempts a destructive action (file delete, form submit with unrecognized data); the before-tool-call guardrail blocks it and forces the agent to choose a safer path.
3. A high-impact action triggers a confirmation gate; the task pauses until the user approves or rejects; on approval the action executes; on rejection the agent receives a `USER_REJECTED` signal and may recover.
4. An operator halt fires mid-task; the entity transitions to `HALTED`; the action history is preserved for audit.

## License

Apache 2.0.
