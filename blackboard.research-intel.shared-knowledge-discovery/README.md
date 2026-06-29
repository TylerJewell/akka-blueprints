# Akka Sample: Blackboard Knowledge Discovery

Specialist analyst agents read and write to a shared structured knowledge object (the blackboard); a control shell picks which specialist runs next based on current blackboard state. No specialist talks to another directly. Demonstrates the **blackboard** coordination pattern with embedded governance.

## Prerequisites
- Claude Code installed
- Akka plugin enabled (`/akka:setup`)
- Host software: none (runs out of the box)
- Model-provider key option: mock LLM during scaffolding, or a real key supplied by env var, env file, secrets URI, or one-time typed value (the key is never written to disk by this sample)

## Generate the system
Copy the blueprint and run `/akka:specify @SPEC.md` in Claude Code. The workflow continues through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build`.

## What you'll get
- **ControlShell** — Workflow that inspects the blackboard, picks the next specialist to invoke, and commits each write under a schema guardrail
- **SourceScout** — AutonomousAgent proposing new sources for the open inquiry
- **ClaimExtractor** — AutonomousAgent turning a source into structured claims
- **HypothesisSynthesizer** — AutonomousAgent proposing a hypothesis from current claims
- **Critic** — AutonomousAgent challenging the weakest claim or hypothesis
- **BlackboardEntity** — One EventSourcedEntity per inquiry; the shared structured state
- **InquiryEntity**, **IntakeQueue** — Inquiry lifecycle and submission log
- **SystemControl** — Operator halt flag
- **BlackboardView** — Read-side projection of every inquiry's current blackboard
- **KnowledgeEndpoint + AppEndpoint** — REST, SSE, and embedded UI
- 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI

## Customise before generating
- `SPEC.md §3` — Change canned inquiries or remove the simulator
- `SPEC.md §11` — Change the specialist roster (e.g., add a NumericChecker)
- `prompts/*.md` — Narrow domain (e.g., biomedical, legal, market research)
- `eval-matrix.yaml` — Extend the schema-validation rules or contribution-utility scoring

## What gets validated
Five user journeys define the acceptance bar:
1. Submit an inquiry → control shell runs specialists → blackboard converges → report rendered
2. Malformed write → schema guardrail rejects → blackboard untouched
3. Low-utility contribution → eval-event records a low score → control shell stops invoking that specialist
4. Critic challenge → flagged item moved to disputed list → control shell reroutes
5. Operator halt → control shell pauses; resume continues the inquiry

**License:** Apache 2.0
