# Architecture — web-navigation-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call per action step. `SessionEndpoint` accepts a task goal, starts a `NavigationWorkflow`, and writes a `SessionStarted` event onto `SessionEntity`. The workflow's `initStep` loads the starting URL via `HeadlessBrowserClient` and captures the first screenshot. On each `actionStep`, the workflow polls `HaltController` first; if the halt flag is set, the workflow branches immediately to `haltedStep` and terminates the session without touching the agent. If not halted, the workflow calls `WebNavigatorAgent` — the single AutonomousAgent — with the goal and the current screenshot as a task attachment. The agent's `before-tool-call` guardrail (`ActionGuardrail`) intercepts each candidate `BrowserAction` before execution, blocking disallowed domains and forbidden actions, and routing high-stakes actions to `hitlStep`. Once an action passes the guardrail and any HITL gate, the workflow executes it via `HeadlessBrowserClient`, records the outcome on `SessionEntity`, and the `ScreenshotConsumer` captures the resulting page state. `SessionView` projects every entity event into a read-model row; `SessionEndpoint` serves the read model over REST and SSE.

Two EventSourcedEntities exist side by side: `SessionEntity` (one per session) and `HaltController` (global singleton). This separation keeps halt state independent of any individual session — clearing the halt flag affects all future action steps across all running sessions without touching their individual session records.

## Interaction sequence

The sequence traces the happy path (J1) with no HITL. Two governance checkpoints occur on every action step:

1. The `HaltController.getHalt()` read — a fast entity read that adds negligible latency but provides a guaranteed pre-execution stop point.
2. The `ActionGuardrail.before-tool-call` check — runs synchronously on the candidate action before any browser interaction. The guardrail does not make an LLM call; it applies deterministic rules.

The screenshot capture is handled asynchronously by `ScreenshotConsumer` after each `ActionExecuted` event, so the workflow does not block on screenshot I/O during the action loop.

## State machine

Seven states. The interesting paths:

- The happy path is `STARTING → NAVIGATING → COMPLETED`.
- The HITL path is `NAVIGATING → AWAITING_APPROVAL → NAVIGATING → ... → COMPLETED`. A HITL rejection returns the session to `NAVIGATING` after the agent replans; an approved action also returns to `NAVIGATING` after execution.
- Two halt transitions: `NAVIGATING → HALTED` and `AWAITING_APPROVAL → HALTED`. The halt check runs before each action step, so a halted session waiting for HITL approval also stops when the workflow resumes.
- `FAILED` is reached on agent error (after retry budget exhaustion) or HITL timeout. Prior session data — action log, screenshots, partial outcome — is preserved on the entity.
- There is no `ROLLBACK` state. Browser actions that already executed cannot be undone by the system; the HITL gate exists precisely to prevent irreversible actions from executing without human confirmation.

## Entity model

`SessionEntity` is the source of truth for one navigation session. It emits ten event types covering goal ingestion, per-action recording, HITL lifecycle, and terminal transitions. `SessionView` projects every event into a row. `ScreenshotConsumer` subscribes to `ActionExecuted` events to feed screenshots back into the loop. `NavigationWorkflow` both reads (`getSession`) and writes (`markNavigating`, `recordAction`, `requestHitl`, `resolveHitl`, `complete`, `halt`, `fail`) on the entity.

`HaltController` is independent of `SessionEntity`. It holds a single global `halted` flag persisted as events (`HaltSet`, `HaltCleared`). Multiple sessions read it; none write to it except through the operator API.

## Defence-in-depth governance flow

For any browser action that executes in this system, it passed through:

1. **HaltController check** — operator can stop all sessions before any agent call starts.
2. **WebNavigatorAgent** — one model call, one typed action output.
3. **before-tool-call guardrail** — domain allowlist and forbidden-action checks block the action before browser interaction.
4. **HITL gate** (conditional) — high-stakes actions pause for human confirmation before the browser executes them.

Each step is independent. A compromised agent that returns a disallowed-domain action is blocked by the guardrail regardless of the agent's rationale. An action that passes the guardrail but is flagged high-stakes is still stopped until a human approves it.
