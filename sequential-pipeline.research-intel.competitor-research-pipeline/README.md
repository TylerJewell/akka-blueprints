# Akka Sample: Competitor Research Pipeline

A single `ResearchAgent` walks a competitor name through three task phases — **SEARCH → SUMMARIZE → PUBLISH** — wired together by explicit task dependencies. The SEARCH phase pulls raw web results via Exa.ai, the SUMMARIZE phase condenses findings into structured profile fields, and the PUBLISH phase writes a typed `CompetitorProfile` record into Notion. Each phase has its own typed input, typed output, and a small set of phase-specific tools. The user submits a competitor name and receives a durable profile card in Notion.

Demonstrates the **sequential-pipeline** coordination pattern wired with one governance mechanism: a `before-tool-call` guardrail that validates every write-to-Notion call against schema constraints and scope rules before the write executes.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- **Exa.ai API key** — required for the live SEARCH phase. Supply it via the same key-sourcing mechanism as the LLM key (env var `EXA_API_KEY`, env file, secrets-store URI, or one-time prompt). The mock LLM path replaces Exa.ai calls with in-process fixture data, so no Exa key is needed in mock mode.
- **Notion integration token and target database ID** — required for the live PUBLISH phase. Supply the token via `NOTION_API_TOKEN` and the database ID via `NOTION_DATABASE_ID` using the same key-sourcing mechanism. In mock mode these are bypassed; the mock publisher writes profiles to an in-process store instead.

## Generate the system

```sh
cp -r ./sequential-pipeline.research-intel.competitor-research-pipeline  ~/my-projects/competitor-research-pipeline
cd ~/my-projects/competitor-research-pipeline
```

(Optional) Edit `SPEC.md` to point at a different Notion database schema, a different Exa.ai search configuration, or a richer set of profile fields.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ResearchAgent** — one AutonomousAgent declaring three Task constants (`SEARCH_COMPETITOR`, `SUMMARIZE_FINDINGS`, `PUBLISH_PROFILE`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **CompetitorPipelineWorkflow** — runs `searchStep → summarizeStep → publishStep → evalStep`. Each step calls `runSingleTask` and writes the typed result back onto `CompetitorEntity` before the next step starts.
- **CompetitorEntity** — an EventSourcedEntity holding the per-competitor lifecycle (`ResultsFetched`, `SummaryProduced`, `ProfilePublished`, `PublishEvaluated`).
- **SearchTools / SummarizeTools / PublishTools** — three function-tool classes registered on the agent, one per phase. The `before-tool-call` guardrail enforces that each tool is only callable in its own phase and that the PUBLISH phase's Notion write conforms to the required schema and scope.
- **NotionWriteGuardrail** — the runtime check that validates every Notion write. A write whose payload is missing required fields, references an out-of-scope database, or carries a field type mismatch is rejected before the write executes.
- **PublishQualityScorer** — deterministic, rule-based on-decision evaluator that runs immediately after `ProfilePublished` and emits a 1–5 completeness score.
- **CompetitorView + CompetitorEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded competitor list under `src/main/resources/sample-events/competitors.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/research-agent.md` — narrow the agent's role (e.g., constrain it to SaaS vendors, to open-source projects, to regional fintech players) by tightening the system prompt and renaming the typed records (`CompetitorProfile`, `FindingSet`, `ProfileSummary`).
- `SPEC.md §5` — extend the typed outputs (`SearchResultSet`, `ProfileSummary`, `CompetitorProfile`) with domain-specific fields. The Notion write guardrail does not need editing unless the Notion schema changes — it validates against the declared `NotionSchema` constants, not field names.
- `eval-matrix.yaml` — wire a real Notion schema check (replace the deterministic stub with a live `retrieve database` call to confirm field names and types) by editing the `G1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a competitor name → `SEARCH` runs → `SUMMARIZE` runs → `PUBLISH` runs → a typed `CompetitorProfile` appears in Notion and in the UI within ~90 s. Every transition is visible in real time.
2. A PUBLISH-phase tool is called while `SummaryProduced` has not yet been recorded (forced via the mock LLM) → the `before-tool-call` guardrail rejects the call → the workflow records the rejection event → the agent retries in-phase → the pipeline completes correctly.
3. A Notion write whose payload is missing the `pricing_model` required field is caught by `NotionWriteGuardrail` before the write executes → rejected with a structured schema-violation error → the agent re-forms the payload → the corrected write proceeds.
4. Each task receives only its own typed inputs; the SEARCH task does not see the Notion publishing instructions, and the PUBLISH task does not see raw web excerpts — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
