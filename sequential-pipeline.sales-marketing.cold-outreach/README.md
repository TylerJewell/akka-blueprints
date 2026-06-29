# Akka Sample: LeadPilot Lite - Personalized Cold Email

A single `OutreachAgent` walks a prospect through three task phases — **RESEARCH → DRAFT → SEND** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a small set of phase-specific tools. The user submits a prospect record and receives a ready-to-send or approved `OutreachEmail`.

Demonstrates the **sequential-pipeline** coordination pattern wired with three governance mechanisms: a `before-tool-call` guardrail that blocks the send tool when the draft has not yet passed content compliance, a `before-agent-response` guardrail that runs CAN-SPAM/GDPR compliance checks on every generated email body before the agent returns it, and an application-layer human-in-the-loop (HITL) approval gate that halts the pipeline until a reviewer explicitly approves or rejects the draft before any send is attempted.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every research / draft / compliance tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.sales-marketing.cold-outreach  ~/my-projects/cold-outreach
cd ~/my-projects/cold-outreach
```

(Optional) Edit `SPEC.md` to point at a different prospect seed list, a different model provider, or tighter compliance rules for your jurisdiction.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **OutreachAgent** — one AutonomousAgent declaring three Task constants (`RESEARCH_PROSPECT`, `DRAFT_EMAIL`, `SEND_EMAIL`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **OutreachPipelineWorkflow** — runs `researchStep → draftStep → approvalStep → sendStep`. Each LLM-calling step calls `runSingleTask` and writes the typed result back onto `ProspectEntity` before the next step starts. `approvalStep` is a synchronous wait on a human decision before `sendStep` proceeds.
- **ProspectEntity** — an EventSourcedEntity holding the per-prospect outreach lifecycle (`ProspectResearched`, `EmailDrafted`, `ComplianceChecked`, `ReviewRequested`, `ReviewDecided`, `EmailSent`).
- **ResearchTools / DraftTools / SendTools** — three function-tool classes registered on the agent, one per phase. The `before-tool-call` guardrail enforces that the send tool is only callable after approval.
- **SendGuardrail** — the `before-tool-call` hook that blocks `sendEmail` unless the entity records an approved review decision. Prevents any send that bypasses the approval gate.
- **ComplianceGuardrail** — the `before-agent-response` hook that runs a deterministic rule-based CAN-SPAM/GDPR check on every email body the agent proposes to return. Rejects drafts that lack an unsubscribe notice, that address a region-blocked contact, or that omit the sender's physical address.
- **ProspectView + OutreachEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded prospect list under `src/main/resources/sample-events/prospects.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/outreach-agent.md` — narrow the agent's role (e.g., constrain it to technical buyers, to a specific industry vertical, or to a single product line) by tightening the system prompt and renaming the typed records (`ProspectProfile`, `EmailDraft`, `OutreachEmail`).
- `SPEC.md §5` — extend the typed outputs with additional personalization fields (e.g., `recentFunding`, `techStack`, `hiringSignals`). The send guardrail does not need editing — it checks the approval decision, not field shapes.
- `eval-matrix.yaml` — wire a real compliance evaluator (replace the deterministic stub with a regex-and-rule checker against your jurisdiction's opt-out requirements) by editing the `H2` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a prospect → `RESEARCH` runs → `DRAFT` runs → the draft halts at the approval gate → the reviewer approves → `SEND` runs → a typed `OutreachEmail` record lands in the UI. Every transition is visible in real time.
2. The agent attempts to call `sendEmail` before the prospect has an approved review decision (forced via the mock LLM) → the `before-tool-call` guardrail rejects the call → the workflow records the rejection event → the agent retries → the pipeline pauses correctly at `approvalStep`.
3. The agent produces an email body that lacks an unsubscribe notice → the `before-agent-response` compliance guardrail rejects it → the agent revises the draft → the revised body passes.
4. A reviewer rejects the draft → the entity transitions to `REVIEW_REJECTED` → the pipeline ends without sending → the UI shows the rejection reason.

## License

Apache 2.0.
