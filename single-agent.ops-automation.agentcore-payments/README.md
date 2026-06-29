# Akka Sample: Agent-Initiated Payment Settlement

An Akka-powered payment settlement agent that autonomously pays external APIs within a configurable spend envelope — with human approval gates for above-threshold payments and automatic halting when the task budget is exhausted.

## Prerequisites

Before running this sample, ensure you have:

1. **Claude Code** with the Akka plugin installed
2. **LLM API key** — choose one approach:
   - **Mock LLM** (no key needed): Use the built-in stub for offline development
   - **Environment variable**: `export ANTHROPIC_API_KEY=<your-key>`
   - **`.env` file**: Create `.env` in the project root with `ANTHROPIC_API_KEY=<your-key>`
   - **Secrets URI**: Configure `akka.secrets.anthropic-api-key` in `application.conf`
   - **Type-once prompt**: The agent endpoint will prompt for the key on first call if none is set
3. **Java 21+** and **Maven 3.9+**
4. **Akka CLI** (`akka` command) — install from [akka.io](https://akka.io)

## Quick start

```bash
# 1. Start the service locally
mvn compile exec:java

# 2. Submit a payment task with a spend envelope
curl -X POST http://localhost:9164/tasks \
  -H 'Content-Type: application/json' \
  -d '{"taskId":"task-001","description":"Fetch enriched firmographic data for 5 accounts","budgetCents":5000,"approvalThresholdCents":1000}'

# 3. Stream task progress
curl http://localhost:9164/tasks/task-001/stream

# 4. Approve a payment held for review
curl -X POST http://localhost:9164/tasks/task-001/approvals \
  -H 'Content-Type: application/json' \
  -d '{"paymentId":"pay-001","decision":"APPROVE","reviewer":"ops@example.com"}'
```

## Key capabilities

- **Autonomous payment decisions** — the agent selects which external API calls to make and at what cost, within the task description
- **Spend envelope enforcement** — every task carries a budget; accumulated spend is tracked in real time and the agent halts immediately on breach
- **Above-threshold approval** — payments that exceed the configured per-payment threshold are held and routed to a human reviewer before execution
- **Immutable audit trail** — every payment decision, approval, rejection, and halt event is persisted as an event-sourced record
- **Live spend ledger** — a read model exposes current spend, remaining budget, and per-payment status for each task

## Run `/akka:specify` to get started

Open this folder in Claude Code and run:

```
/akka:specify
```

Claude will read `SPEC.md` and generate a fully working Akka service.
