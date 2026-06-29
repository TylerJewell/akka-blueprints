# Akka Sample: Modular Agent Skills Loader

This sample demonstrates how to build a single-agent system where skills are composable folders loaded on demand at runtime. An agent queries a skill registry to select the right skill for each incoming task, and a guardrail fires before any tool invocation to confirm the selected capability is present and enabled in the registry.

## What it demonstrates

- **Skill registry** — a Key-Value Entity that stores skill descriptors (name, version, capabilities, enabled flag).
- **Runtime skill selection** — the AI agent queries the registry at task dispatch time rather than hardcoding skill bindings.
- **Guardrail before tool invocation** — every tool call is checked against the registry; disabled or unregistered capabilities are blocked before execution begins.
- **Workflow orchestration** — a Workflow component coordinates the validate → load → verify → execute lifecycle and persists each state transition as an event.
- **SSE task stream** — callers can subscribe to a server-sent-events stream to receive live task-status updates.

## Prerequisites

- **Claude Code** with the Akka plugin installed.
- **AI model key** — choose one approach (never write the key value to disk):
  - Mock LLM: no key required; set `MOCK_LLM=true`.
  - Environment variable: export `ANTHROPIC_API_KEY` (or your provider's variable) in your shell before running.
  - Env file: create a `.env` file (gitignored) containing the variable.
  - Secrets URI: configure `akka.secrets.uri` in `application.conf` to point to a secrets manager.
  - Type-once: the Akka local console prompts for the key on first run and holds it in memory.
- **Java 21+** and **Maven 3.9+** (required once `/akka:implement` generates sources).
- **Akka CLI** (`akka`) for local runtime management.

## Quick start

```bash
# 1. Open this folder in Claude Code
cd akka-blueprints/single-agent.general.agent-skills

# 2. Run /akka:specify to turn SPEC.md into a full feature spec
# (Claude Code + Akka plugin required)

# 3. Run /akka:plan  → /akka:tasks  → /akka:implement
# Each step generates artifacts and waits for your review.

# 4. Run /akka:build to compile and test locally.

# 5. Start the local runtime (port 9958):
#    akka local run --port 9958
```

## Repository layout

```
single-agent.general.agent-skills/
├── README.md                        # this file
├── SPEC.md                          # feature specification (12 + 1 sections)
├── PLAN.md                          # architecture diagrams (4 Mermaid)
├── eval-matrix.yaml                 # control definitions
├── risk-survey.yaml                 # deployer risk survey template
├── .corpus-stamp                    # content fingerprint
├── prompts/
│   └── task-dispatch-agent.md       # AI agent system prompt
└── reference/
    ├── architecture.md              # narrative architecture guide
    ├── api-contract.md              # full endpoint + SSE reference
    ├── ui-mockup.md                 # five-tab HTML mockup
    ├── user-journeys.md             # acceptance journeys
    └── data-model.md                # records, events, enums
```

## Further reading

- [SPEC.md](SPEC.md) — full feature specification.
- [reference/architecture.md](reference/architecture.md) — component narrative.
- [reference/api-contract.md](reference/api-contract.md) — REST + SSE contract.
- [reference/user-journeys.md](reference/user-journeys.md) — acceptance journeys.
- [eval-matrix.yaml](eval-matrix.yaml) — guardrail control definition.
