# Akka Sample: Autonomous Agent

A single coding agent reads a GitHub issue, plans a fix, executes shell and file-system tools in a loop, and emits a structured `TaskOutcome` — patch diff, confidence score, and a per-step execution trace. Every tool call is gated before it runs; a human checkpoint approves the proposed patch before any permanent write; and a deployer-controlled kill switch can halt the loop at any point.

Demonstrates the **single-agent** coordination pattern in the software-development domain, wired with four governance mechanisms: a before-tool-call guardrail on every shell and file write, an operator kill switch, an application-level human checkpoint, and deployer-runtime monitoring over the full execution loop.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- A valid credential for the GitHub API (a fine-grained personal access token with `repo` read scope on the target repository). Supply it the same way as the model-provider key: existing env var (`GITHUB_TOKEN`), env file, secrets-store URI, or one-time session prompt. Akka records only the reference — never the value.

## Generate the system

```sh
cp -r ./single-agent.dev-code.autonomous-agent-pattern  ~/my-projects/autonomous-agent
cd ~/my-projects/autonomous-agent
```

(Optional) Edit `SPEC.md` to point at a different repository or adjust the set of tools the agent is permitted to call (e.g., restrict to file-read-only for a read-only audit mode, or add a `grep-codebase` tool for larger repos).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CodingAgent** — an AutonomousAgent that receives a GitHub issue as a task attachment, plans a fix, and calls tools (read file, write file, run shell command, search codebase) in a loop. Returns a typed `TaskOutcome`.
- **AgentTaskWorkflow** — orchestrates checkpoint-wait → execute → monitor per submitted issue. Includes a human-checkpoint step that pauses the loop pending approval.
- **AgentTaskEntity** — an EventSourcedEntity holding the per-task lifecycle.
- **ToolCallGuardrail** — a before-tool-call guardrail that runs before every shell command and file write, blocking disallowed operations.
- **RuntimeMonitor** — a Consumer that subscribes to `ToolCallExecuted` events and emits alert events when anomalous patterns appear in the agent's tool usage.
- **AgentTaskView + AgentTaskEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded GitHub issues for your own repository's issues (the JSONL file under `src/main/resources/sample-events/seed-issues.jsonl` after generation).
- `SPEC.md §5` — extend `TaskOutcome` with project-specific fields (e.g., `testResults`, `ciPipelineId`, `reviewers`).
- `prompts/coding-agent.md` — narrow the agent's role (a security-focused deployer would restrict the agent to read-only tools and require HITL approval for every proposed write).
- `eval-matrix.yaml` — wire a real sandboxed execution environment by naming it under the guardrail mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a GitHub issue → the agent plans a fix → executes tools → the proposed patch appears pending human approval.
2. The agent attempts a disallowed tool call (e.g., `rm -rf`) → the before-tool-call guardrail blocks it → the agent receives the rejection and proposes an alternative.
3. The operator calls the kill-switch endpoint → the running agent loop halts within its next iteration timeout → the entity transitions to `HALTED`.
4. The RuntimeMonitor detects an anomalous tool-call pattern (e.g., more than 10 shell commands in one minute) and emits a `MonitorAlert` event visible on the UI's monitoring panel.

## License

Apache 2.0.
