# Akka Sample: Valuation Reviewer

An agent-driven workflow that **drafts, critiques, and refines** a valuation report through an iterative critique-refine loop — all within a single stateful Akka workflow. The drafter agent produces an initial valuation, the critic agent checks it against comparable transactions and review standards, failing checks trigger targeted revisions, and the cycle repeats until every check passes or the iteration budget is exhausted. A human reviewer signs off before the report is released.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| Claude Code + Akka plugin | Run `/akka:setup` to install both |
| JDK 21+ | Tested with Eclipse Temurin 21 |
| Maven 3.9+ | Included in the dev container |
| An AI key | See options below |

### AI key options (choose one)

| Option | How |
|---|---|
| Mock LLM (no key) | Set `MOCK_LLM=true` in your env — all LLM calls return canned responses |
| Environment variable | `export ANTHROPIC_API_KEY=<your-key>` |
| `.env` file | Create `.env` in project root: `ANTHROPIC_API_KEY=<your-key>` |
| Secrets URI | `akka secret create anthropic-key --literal ANTHROPIC_API_KEY=<your-key>` |
| Type once at runtime | The service prompts on first request if no key is configured |

---

## Quick start

```bash
# 1 – generate the implementation
/akka:specify   # reads SPEC.md, emits questions, locks design

# 2 – build and test locally
/akka:build

# 3 – run locally
/akka:deploy --local
```

---

## What gets generated

| Artifact | Purpose |
|---|---|
| `ValuationReviewWorkflow` | Stateful loop — drives draft → critique → revise cycles |
| `ValuationDrafterAction` | LLM agent that writes and revises valuation report sections |
| `ValuationCriticAction` | LLM agent that checks the report against comps and review standards |
| `ValuationReportView` | Read-side KV view of current report state |
| `ValuationEndpoint` | HTTP + SSE API surface |

---

## Governance at a glance

| Control | Kind | Enforcement |
|---|---|---|
| Human sign-off | HITL — application gate | Workflow pauses; resumes only on explicit `POST /sign-off` |
| Iteration cap | HALT — budget cap | Workflow terminates after N cycles, emits `IterationCapReached` event |
