# Akka Sample: AI Blog Writer Pipeline with Ollama

A single `BlogWriterAgent` walks a topic through five task phases — **RESEARCH → OUTLINE → DRAFT → EDIT → PUBLISH** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a small set of phase-specific tools. The user submits a topic and receives a structured `Post`.

Demonstrates the **sequential-pipeline** coordination pattern wired with one governance mechanism: a `before-agent-response` guardrail that blocks the agent's response from advancing to the next phase until a content-policy and originality check clears the prose.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The pipeline runs out of the box — every research, outline, draft, edit, and publish tool is implemented in-process inside the same Akka service. Ollama is the default model provider when running locally; the service falls back gracefully to the configured env-var key if Ollama is not present.

## Generate the system

```sh
cp -r ./sequential-pipeline.content-editorial.blog-pipeline  ~/my-projects/blog-pipeline
cd ~/my-projects/blog-pipeline
```

(Optional) Edit `SPEC.md` to point at a different seeded topic list, a different model provider, or a richer set of editorial tools.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **BlogWriterAgent** — one AutonomousAgent declaring five Task constants (`RESEARCH_TOPIC`, `OUTLINE_POST`, `DRAFT_POST`, `EDIT_POST`, `PUBLISH_POST`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **BlogPipelineWorkflow** — runs `researchStep → outlineStep → draftStep → editStep → publishStep → guardStep`. Each step calls `runSingleTask` and writes the typed result back onto `PostEntity` before the next step starts.
- **PostEntity** — an EventSourcedEntity holding the per-post lifecycle (`ResearchCollected`, `OutlineProduced`, `DraftWritten`, `EditApplied`, `ContentCleared`, `PostPublished`, `GuardrailBlocked`, `PostFailed`).
- **ResearchTools / OutlineTools / DraftTools / EditTools / PublishTools** — five function-tool classes registered on the agent, one per phase. The `before-agent-response` guardrail checks the final prose of each phase before the result is written to the entity.
- **ContentPolicyGuardrail** — the runtime check that backs the content-policy and originality contract. A phase response whose prose fails the policy check is rejected; the workflow records a `GuardrailBlocked` event and the agent retries.
- **PostView + PostEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded topic set under `src/main/resources/sample-events/topics.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/blog-writer-agent.md` — narrow the agent's role (e.g., constrain it to technical tutorials, product announcements, or opinion pieces) by tightening the system prompt and renaming the typed records (`Outline`, `Draft`, `Post`).
- `SPEC.md §5` — extend the typed outputs (`ResearchNotes`, `Outline`, `Draft`, `EditedDraft`, `Post`) with editorial-specific fields.
- `eval-matrix.yaml` — wire a real policy engine (replace the keyword-based stub with an embedding-similarity check) by editing the `G1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a topic → `RESEARCH` runs → `OUTLINE` runs → `DRAFT` runs → `EDIT` runs → `PUBLISH` runs → a typed `Post` lands in the UI within ~90 s. Every transition is visible in real time.
2. A draft whose prose contains a flagged keyword (forced via the mock LLM) fails the `before-agent-response` guardrail → the workflow records a `GuardrailBlocked` event → the agent retries → the pipeline completes correctly.
3. Every `Post` emitted has a policy-clear status visible on the same UI card; posts that exceeded the retry budget land in `FAILED` with the last block reason visible.
4. Each task receives only its own typed inputs; the RESEARCH task does not see draft instructions, and the PUBLISH task does not see raw research notes — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
