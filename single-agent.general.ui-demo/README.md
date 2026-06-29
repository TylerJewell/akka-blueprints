# Akka Sample: AG-UI + CopilotKit Integration

A single copilot agent accepts a user prompt through a streaming AG-UI/CopilotKit-compatible front end and returns a structured response with inline citations. The agent streams token-by-token output; a `before-agent-response` guardrail validates the final structured payload before it is committed to the session log.

Demonstrates the **single-agent** coordination pattern wired to a streaming-UI integration layer. The UI speaks the AG-UI protocol (Server-Sent Events with typed message envelopes); CopilotKit's front-end hooks consume those envelopes directly. One guardrail sits at the agent boundary to keep customer-facing output well-formed.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the copilot session store lives in-process, and the AG-UI/CopilotKit wire protocol is implemented entirely in the generated static UI.

## Generate the system

```sh
cp -r ./single-agent.general.ui-demo  ~/my-projects/ui-demo
cd ~/my-projects/ui-demo
```

(Optional) Edit `SPEC.md` to change the seeded prompt library or to swap the default response schema (e.g., add a `sourceUrls` field or a `confidenceScore`).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CopilotAgent** — an AutonomousAgent that accepts a user prompt and context, and returns a typed `CopilotResponse` with an answer, inline citations, and a suggested follow-up prompt.
- **SessionWorkflow** — orchestrates prompt submission → streaming agent call → response commit per session turn.
- **SessionEntity** — an EventSourcedEntity holding per-session turn history.
- **ResponseGuardrail** — a `before-agent-response` hook that validates the structured output before it leaves the agent loop.
- **SessionView + CopilotEndpoint + AppEndpoint** — read model, AG-UI-compatible REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded prompt library under `src/main/resources/sample-events/seed-prompts.jsonl`.
- `SPEC.md §5` — extend `CopilotResponse` with domain-specific fields (e.g., `productSku`, `ticketId`, `workflowAction`).
- `prompts/copilot-agent.md` — narrow the agent's role (an e-commerce deployer would constrain it to product queries; a support deployer to ticket triage).
- `eval-matrix.yaml` — add a `regulation_anchors` entry if the deployer's jurisdiction has AI transparency obligations.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a prompt → the session turn appears streaming → the final structured response lands in the UI.
2. The agent's first response is malformed (mock LLM path) → the `before-agent-response` guardrail rejects it → the agent retries → a well-formed response lands.
3. Every turn's response is confirmed in the session history; re-opening the session shows the full history.
4. An over-length response (exceeds `maxTokens`) is caught by the guardrail's length check before reaching the UI.

## License

Apache 2.0.
