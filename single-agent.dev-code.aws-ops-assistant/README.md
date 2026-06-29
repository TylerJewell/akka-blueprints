# Akka Sample: AWS Ops Assistant (MCP)

A single-agent AWS console assistant that accepts natural-language operations requests, translates them into MCP-exposed AWS API calls, and returns a structured operation report. Every mutating call (create, update, delete, attach, detach) is gated by a confirmation step before execution; a guardrail validates that the requested action targets an allowed resource type; and an emergency-stop control lets an operator halt the agent mid-flight when a wide-blast-radius action is in progress.

Demonstrates the **single-agent** coordination pattern wired with three governance mechanisms: a `before-tool-call` guardrail that blocks disallowed resource targets before any AWS API is called, an application-level human-in-the-loop that requires explicit user confirmation before mutating calls execute, and an operator halt that can freeze the running agent on demand.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The AWS MCP surface is simulated in-process; no real AWS account or credentials are required.

## Generate the system

```sh
cp -r ./single-agent.dev-code.aws-ops-assistant  ~/my-projects/aws-ops-assistant
cd ~/my-projects/aws-ops-assistant
```

(Optional) Edit `SPEC.md` to point the agent at a specific set of allowed AWS resource types (e.g., restrict to EC2 + S3 only, or expand to include RDS and Lambda).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **AwsOpsAgent** — an AutonomousAgent that accepts an operations request in natural language, calls MCP-exposed AWS read/write tools, and returns a structured `OperationReport`.
- **OpsWorkflow** — orchestrates request-intake → guardrail-check → confirmation-wait → execution → report per submitted request.
- **OpsRequestEntity** — an EventSourcedEntity holding the per-request lifecycle from SUBMITTED through COMPLETED or HALTED.
- **ConfirmationConsumer** — a Consumer that watches for `ConfirmationRequested` events and routes them to the UI confirmation queue.
- **OpsView + OpsEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — adjust the seeded operation scripts (the JSONL file under `src/main/resources/sample-events/operation-scripts.jsonl` after generation).
- `SPEC.md §5` — extend `OperationReport` with service-specific fields (e.g., `affectedArns`, `estimatedCost`, `changeTicketId`).
- `prompts/aws-ops-agent.md` — narrow the agent's allowed resource scope (restrict to a specific region, a specific account, or a specific tag namespace).
- `eval-matrix.yaml` — wire the guardrail's `allowedResourceTypes` list to a deployer-specific policy file.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a read-only request → the agent queries AWS APIs → a report appears in the UI without any confirmation prompt.
2. A user submits a mutating request → the `before-tool-call` guardrail verifies the resource type is allowed → the UI shows a confirmation card → the user approves → the agent executes → the report appears.
3. An operator halts an in-flight agent; the request transitions to HALTED and no further AWS calls are made.
4. A mutating call targeting a resource type outside the allow-list is rejected by the guardrail; the agent returns an `OperationReport` with status BLOCKED without making any AWS API call.

## License

Apache 2.0.
