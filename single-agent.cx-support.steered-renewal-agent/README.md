# Akka Sample: Library Book Renewal (Steering)

A single renewal-agent accepts a patron's book-renewal request and produces a structured renewal decision: APPROVED / DENIED / EXTENDED with a reason. The request is processed by one AI agent; a `before-agent-response` guardrail enforces library renewal policy before the decision leaves the agent loop, demonstrating **steering** as a runtime governance pattern.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — patron records and loan state live in-process and no external library management system is required.

## Generate the system

```sh
cp -r ./single-agent.cx-support.steered-renewal-agent  ~/my-projects/steered-renewal-agent
cd ~/my-projects/steered-renewal-agent
```

(Optional) Edit `SPEC.md` to adjust the renewal policy (e.g., extend the maximum renewal window, add a fine-threshold above which renewals are blocked, or change the patron tiers that qualify for automatic approval).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **RenewalAgent** — an AutonomousAgent that accepts a renewal request (patron record + current loan) as a task and returns a typed `RenewalDecision`.
- **RenewalWorkflow** — orchestrates enrich-wait → decide → notify per submitted renewal.
- **LoanEntity** — an EventSourcedEntity holding the per-loan lifecycle.
- **PolicyEnforcer** — a `before-agent-response` guardrail that validates every candidate decision against the library's renewal policy before it leaves the agent loop.
- **RenewalView + RenewalEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded patron tiers and loan scenarios (the JSONL file under `src/main/resources/sample-events/seed-loans.jsonl` after generation).
- `SPEC.md §5` — add fields to `RenewalDecision` (e.g., `newDueDate`, `renewalCount`, `patronTier`).
- `prompts/renewal-agent.md` — tighten the agent's scope (an academic library deployer might restrict renewals to items with fewer than three prior renewals; a public library might allow indefinite renewal for patrons with no outstanding fines).
- `eval-matrix.yaml` — wire the guardrail to a real policy-rules engine by naming it under the `PolicyEnforcer` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A patron submits a renewal request → the agent decides → the decision appears in the UI with the outcome and reason.
2. The agent returns a decision that violates a hard policy rule (e.g., approves a renewal for a patron above the fine threshold) → the `before-agent-response` guardrail rejects it → the agent retries → a compliant decision lands.
3. A renewal request for an item already at the maximum renewal count is denied with a clear reason, never approved.
4. The live list updates in real time via SSE as each renewal transitions through its lifecycle states.

## License

Apache 2.0.
