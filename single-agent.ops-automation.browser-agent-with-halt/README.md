# Akka Sample: Web Navigation Agent (WebVoyager)

A vision-language agent that browses real websites by observing screenshots and issuing click, type, and scroll actions to complete tasks. The agent reads the rendered page, decides the next action, and executes it until the task goal is reached or the operator halts it.

Demonstrates the **single-agent** coordination pattern wired with three governance mechanisms: a `before-tool-call` guardrail that screens every browser action before execution, an operator/regulator halt switch that immediately stops any in-flight session, and a human-in-the-loop approval gate that pauses high-stakes actions (form submit, purchase confirmation, account changes) until a human approves or rejects them.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key that supports vision inputs — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The browser automation runs via a headless browser bundled in-process using Playwright; no separate browser daemon or external service is required.

## Generate the system

```sh
cp -r ./single-agent.ops-automation.browser-agent-with-halt  ~/my-projects/web-navigation-agent
cd ~/my-projects/web-navigation-agent
```

(Optional) Edit `SPEC.md` to narrow the allowed-domain list (e.g., restrict to an internal portal), change the HITL approver configuration, or swap the seeded tasks.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **WebNavigatorAgent** — an AutonomousAgent that receives a task goal and a current-page screenshot as a task attachment, selects the next browser action, and returns a typed `BrowserAction`.
- **NavigationWorkflow** — orchestrates action-loop → halt-check → HITL-pause → action-execute per session, capped at a configurable max-step budget.
- **SessionEntity** — an EventSourcedEntity holding the per-session lifecycle: goal, visited URLs, action history, HITL-pending decisions, terminal outcome.
- **ActionGuardrail** — a `before-tool-call` guardrail that checks each `BrowserAction` against the allowed-domain list, the forbidden-action list, and the high-stakes classifier before execution.
- **HaltController** — an EventSourcedEntity that holds the global halt flag; the workflow polls it before each action step.
- **ScreenshotConsumer** — a Consumer that subscribes to `ActionExecuted` events, captures the resulting page screenshot, and feeds it back as the next agent input attachment.
- **SessionView + SessionEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded task list for your own goals (internal portal navigation, data-entry automation, form-fill workflows).
- `SPEC.md §5` — extend `BrowserAction` with domain-specific action types (e.g., a `file-upload` action, a `captcha-skip` sentinel that routes to HITL automatically).
- `prompts/web-navigator.md` — narrow the agent's behaviour (e.g., add a rule that the agent must never enter payment details it was not explicitly given in the task goal).
- `eval-matrix.yaml` — update `regulation_anchors` for your jurisdiction once the baseline controls are confirmed in production.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a task goal → the agent navigates autonomously → the session reaches `COMPLETED` with a full action log and final screenshot.
2. The operator triggers the halt switch mid-session → the workflow stops within one action step → the session transitions to `HALTED`; no further browser actions execute.
3. The agent selects a form-submit action → the HITL gate fires → the session pauses at `AWAITING_APPROVAL`; a human approves → navigation resumes; a human rejects → the session transitions to `REJECTED`.
4. The agent attempts to navigate to a domain not in the allowed list → the `before-tool-call` guardrail blocks the action and the agent retries with a different approach.

## License

Apache 2.0.
