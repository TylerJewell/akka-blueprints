# Akka Sample: Deterministic Workflow with LLM Activities

A `ContentWorkflow` drives a topic through two sequential LLM-backed activities — **OUTLINE → DRAFT** — with no autonomous agent involved. Each activity is a durable, typed LLM call: the first produces a structured `PostOutline`; the second consumes that outline and returns a finished `BlogPost`. The workflow orchestrates the handoff; the LLM is called twice, in order, with the first call's output as context for the second.

Demonstrates the **sequential-pipeline** coordination pattern expressed as a pure Akka Workflow — no `AutonomousAgent`, no tool loop. Two governance mechanisms are wired: an `after-llm-response` guardrail that schema-validates the structured `PostOutline` before it is passed to the draft activity, and a `before-agent-response` guardrail that reviews the finished `BlogPost` for editorial policy before the post record moves to `READY`.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. Both LLM activities run in-process. The editorial-policy check uses rule-based logic for the mock path.

## Generate the system

```sh
cp -r ./sequential-pipeline.content-editorial.akka-llm-pipeline-workflow  ~/my-projects/llm-pipeline-workflow
cd ~/my-projects/llm-pipeline-workflow
```

(Optional) Edit `SPEC.md` to change the seeded topic list, the target word count, or the editorial policy rules applied by `EditorialGuardrail`.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ContentWorkflow** — one Akka `Workflow` per post request. Two steps: `outlineStep` calls `OutlineActivity` and writes `OutlineProduced` onto `PostEntity`; `draftStep` calls `DraftActivity` with the recorded outline and writes `DraftProduced`; `reviewStep` invokes the editorial guardrail and writes `PostApproved` or `PostFlagged`. A fourth `failStep` records `PostFailed` and ends.
- **OutlineActivity** — a typed LLM call: takes a `topic` and a `wordTarget`, returns a structured `PostOutline`. Registered with `afterLlmResponse` schema-validation guardrail.
- **DraftActivity** — a typed LLM call: takes a `PostOutline`, returns a `BlogPost`. Registered with `beforeAgentResponse` editorial guardrail.
- **PostEntity** — an `EventSourcedEntity` holding the per-post lifecycle (`PostCreated`, `OutlineStarted`, `OutlineProduced`, `DraftStarted`, `DraftProduced`, `PostApproved`, `PostFlagged`, `PostFailed`).
- **SchemaGuardrail** — validates the `PostOutline` returned by the outline LLM call before it advances to the draft step.
- **EditorialGuardrail** — reviews the finished `BlogPost` against editorial rules (length, prohibited topics, required attribution) before the post is marked `READY`.
- **PostView + PostEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded topic list under `src/main/resources/sample-events/topics.jsonl`.
- `SPEC.md §4` and `prompts/outline-activity.md` / `prompts/draft-activity.md` — tune the LLM prompts: target word count, section count, tone, or citation requirements.
- `SPEC.md §5` — extend `PostOutline` with additional fields (e.g., `targetAudience`, `keywordList`). The `SchemaGuardrail` checks field presence — update it to match.
- `eval-matrix.yaml` — swap the rule-based editorial check in `H2` for a real policy engine by editing the `implementation` paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a topic → `outlineStep` runs → `draftStep` runs → the post reaches `READY` in the UI within ~60 s. Both the outline and the final post are visible in real time.
2. A mock-LLM outline that is missing required fields (`sections` empty, `title` blank) is caught by `SchemaGuardrail` at the `after-llm-response` hook; the workflow records the rejection and fails gracefully.
3. A mock-LLM draft that contains a prohibited topic phrase is caught by `EditorialGuardrail` at the `before-agent-response` hook; the post lands in `FLAGGED` state rather than `READY`; the UI card shows the flag reason.
4. The outline produced by `outlineStep` is the only input to `draftStep` — the draft LLM call does not receive the raw topic or any other context beyond the structured outline; the workflow's typed handoff is the only information path between activities.

## License

Apache 2.0.
