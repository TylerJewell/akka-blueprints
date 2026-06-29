# Akka Sample: Content Pipeline Subworkflow Orchestrator

A `ContentPipelineAgent` walks a topic through three task phases — **RESEARCH → DRAFT → REVIEW** — where each phase's tools delegate to dedicated sub-workflows (`ResearchSubworkflow`, `DraftSubworkflow`, `ReviewSubworkflow`). The agent produces a publishable article draft; a `before-agent-response` guardrail checks tone and factuality on the completed draft before it leaves the agent boundary; and a human-in-the-loop approval step gates the publish transition.

Demonstrates the **sequential-pipeline** coordination pattern with sub-workflows as the tool implementation layer: each tool call starts a multi-step sub-workflow and waits for its typed result, so the per-phase capability is itself orchestrated — not a single function call.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. Every research, draft, and review tool executes in-process against sample data under `src/main/resources/sample-data/`.

## Generate the system

```sh
cp -r ./sequential-pipeline.content-editorial.subworkflow-pipeline  ~/my-projects/content-pipeline
cd ~/my-projects/content-pipeline
```

(Optional) Edit `SPEC.md` to change the seeded topic list, tighten the tone-guardrail rules, or extend the review criteria.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ContentPipelineAgent** — one AutonomousAgent declaring three Task constants (`RESEARCH_TOPIC`, `DRAFT_CONTENT`, `REVIEW_DRAFT`); the workflow runs them in order, feeding each typed output forward as the next task's instruction context.
- **ContentPipelineWorkflow** — runs `researchStep → draftStep → reviewStep → guardStep → awaitApprovalStep`. Each agent-calling step delegates to the agent and writes the typed result back onto `ArticleEntity` before advancing. `guardStep` runs `ToneGuardrail` synchronously after `reviewStep` completes. `awaitApprovalStep` pauses the workflow until an editor submits a decision.
- **ResearchSubworkflow / DraftSubworkflow / ReviewSubworkflow** — three Akka Workflows, one per phase. Each sub-workflow is started by a function-tool call on the agent and carries its own multi-step internal execution (gather-sources + extract-facts, build-outline + write-sections, fact-check + tone-score).
- **ResearchTools / DraftTools / ReviewTools** — function-tool classes. Each tool method starts the matching sub-workflow via `componentClient`, waits for its completion, and returns the typed result to the agent. The agent never invokes sub-workflow steps directly.
- **ToneGuardrail** — `before-agent-response` guardrail registered on `ContentPipelineAgent`. Checks every agent response leaving the REVIEW task for off-brand language and unverified factual claims. A flagged response is rejected and the agent retries within its iteration budget.
- **ArticleEntity** — EventSourcedEntity holding the per-article lifecycle and all typed phase outputs.
- **ArticleView + ContentEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded topic set under `src/main/resources/sample-data/topics.jsonl`.
- `SPEC.md §4` and `prompts/content-pipeline-agent.md` — narrow the agent's scope (e.g., constrain to technical documentation, regulatory summaries, product release notes) by tightening the system prompt.
- `SPEC.md §8` — extend the `ToneGuardrail` rules for your brand voice. The guardrail's accept/reject logic is a plain Java class — no LLM call — so the same draft always scores the same.
- `eval-matrix.yaml` — wire a real factuality checker (replace the deterministic stub with a vector-similarity check against your source corpus) by editing the `G1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a topic → RESEARCH runs → DRAFT runs → REVIEW runs → a typed `Draft` lands on the article card with an eval score within ~90 s. Every transition is visible in real time.
2. The agent's REVIEW task response contains an off-brand phrase → `ToneGuardrail` rejects it → the agent retries → the article completes with a clean review.
3. An editor submits a Reject decision on an `AWAITING_APPROVAL` article → the article transitions to `REJECTED`; the UI shows the reason.
4. Each task's tool calls are limited to that phase's sub-workflow tools; no cross-phase sub-workflow is started during the wrong phase.

## License

Apache 2.0.
