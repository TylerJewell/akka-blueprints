# Akka Sample: Blackboard SWE Coordination

A nine-specialist software-engineering team shares code state, tickets, and design decisions on a single blackboard entity. A controller workflow reads the blackboard, determines which specialist has unfinished work, and invokes that agent. A lead-engineer signoff step gates every merge. Demonstrates the **blackboard** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the blackboard, the signoff gate, and the ticket intake are all modelled inside the same service with first-party Akka primitives.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./blackboard.dev-code.swe-coordination  ~/my-projects/blackboard-swe-coordination
cd ~/my-projects/blackboard-swe-coordination
```

(Optional) Edit `SPEC.md` to change the ticket briefs the simulator drips, the specialist roster, or the model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **Nine AutonomousAgents** — Planner, Architect, BackendDev, FrontendDev, Reviewer, Tester, SecurityAnalyst, DocWriter, IntegrationPlanner — each a specialist that reads the blackboard, contributes its findings, and returns a typed result.
- **ControllerWorkflow** — the blackboard controller that reads the current stage, decides which specialist has work to do, invokes that agent, and writes the result back to the blackboard. Iterates until all specialists have contributed.
- **SignoffWorkflow** — pauses for a lead-engineer HITL approval before advancing the blackboard to `MERGED`.
- **BlackboardEntity** — the central knowledge store: tickets, architecture decisions, code artifacts, review findings, test results, security findings, and documentation.
- **TicketEntity**, **SignoffEntity**, **TicketQueue** — ticket lifecycle, signoff records, and the submission audit log.
- **BlackboardView** — the shared read model the controller polls and the UI streams.
- **TicketConsumer** — subscribes to `TicketQueue` events and starts a `ControllerWorkflow` per ticket.
- **TicketSimulator**, **StuckBoardMonitor** — sample ticket drip and stuck-board recovery.
- **BoardEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the ticket briefs the simulator drips, or remove the simulator entirely.
- `SPEC.md §11` — adjust the specialist roster or their tool permissions.
- `prompts/<specialist>.md` — narrow any specialist to a specific language, stack, or style guide.
- `eval-matrix.yaml` — extend the before-state-write guardrail's check list if you add external tool calls.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a ticket → the controller drives all nine specialists → the blackboard reaches `MERGED`.
2. A specialist attempts a type-invalid or lint-failing code write → the guardrail rejects it before the blackboard is updated.
3. All specialists complete → the signoff workflow pauses → a lead-engineer approves via the endpoint → `MERGED`.
4. A lead engineer rejects the merge → the blackboard returns to `IN_REVIEW` for rework.
5. Two controller workflows start concurrently for different tickets → each ticket progresses independently with no cross-contamination on the blackboard.
6. The operator halts the system → the controller stops invoking specialists; resume continues from the last stable stage.

## License

Apache 2.0.
