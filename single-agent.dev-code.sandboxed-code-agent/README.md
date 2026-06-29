# Akka Sample: Sandboxed Code Execution

A single code-execution agent accepts a natural-language task, generates Python to solve it, and runs that code inside an isolated sandbox. Supported sandbox back-ends are Playwright-managed browser automation, E2B cloud microVMs, Modal ephemeral containers, and a local Docker executor. The agent never executes code directly — every tool call is intercepted by a blocking guardrail that validates the code before the sandbox receives it, and a safety-halt mechanism terminates runaway processes.

Demonstrates the **single-agent** coordination pattern with two governance mechanisms: a `before-tool-call` guardrail that blocks dangerous code patterns before any sandbox invocation, and an automatic safety-halt that kills sandbox processes exceeding their resource budget.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- One sandbox integration (choose one):
  - **Playwright** — install via `npm install -g playwright && playwright install chromium`. No API key required.
  - **E2B** — create an account at [e2b.dev](https://e2b.dev/); set `E2B_API_KEY` in your environment.
  - **Modal** — install via `pip install modal` and run `modal setup`. No extra env var beyond Modal's own auth.
  - **Docker** — Docker Engine running locally. No API key required.

  The blueprint defaults to the local Docker executor when no other back-end is configured. Set `SANDBOX_BACKEND=playwright|e2b|modal|docker` to override.

## Generate the system

```sh
cp -r ./single-agent.dev-code.sandboxed-code-agent  ~/my-projects/sandboxed-code-agent
cd ~/my-projects/sandboxed-code-agent
```

(Optional) Edit `SPEC.md` to pre-wire a specific sandbox back-end or to tighten the code-screening rules in the guardrail.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CodeExecutionAgent** — an AutonomousAgent that accepts a task description, generates Python to solve it, calls the `execute_code` tool, and returns a typed `ExecutionResult`.
- **ExecutionWorkflow** — orchestrates submit → screen → execute → record per submitted task.
- **ExecutionEntity** — an EventSourcedEntity holding the per-execution lifecycle.
- **SandboxRouter** — a Consumer that subscribes to `CodeSubmitted` events and routes to the configured back-end (Playwright, E2B, Modal, Docker).
- **CodeScreeningGuardrail** — registered as a `before-tool-call` hook on `CodeExecutionAgent`; blocks forbidden patterns before any sandbox call.
- **SafetyHaltMonitor** — detects runaway processes via the sandbox's resource-usage channel and emits `ExecutionHalted` when limits are exceeded.
- **ExecutionView + ExecutionEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded task catalogue (the JSONL file under `src/main/resources/sample-events/seed-tasks.jsonl` after generation).
- `SPEC.md §5` — extend `ExecutionResult` with domain-specific output fields (e.g., `chartUrl`, `dataframeRows`, `errorCategory`).
- `prompts/code-execution-agent.md` — narrow the agent's capabilities (a data-science deployer would restrict it to pandas/numpy/matplotlib; a web-automation deployer to Playwright-browser tasks only).
- `eval-matrix.yaml` — wire a real static-analysis tool (e.g., Bandit, semgrep) into the guardrail's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a task → the agent generates code → the guardrail approves it → the sandbox runs it → the result appears in the UI within 30 s.
2. The agent generates code containing a forbidden pattern (network exfiltration, host-path traversal) — the `before-tool-call` guardrail blocks it; the agent revises and resubmits; the UI shows only the approved run.
3. A task whose sandbox process exceeds the CPU or wall-clock budget is killed by `SafetyHaltMonitor`; the entity records `ExecutionHalted`; the UI shows a HALTED status card.
4. The LLM call log never contains secrets from the host environment — the sandbox receives only the code string, not the session's env.

## License

Apache 2.0.
