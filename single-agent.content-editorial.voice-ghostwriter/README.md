# Akka Sample: Ghostwriter

A single ghostwriter agent reads a corpus of a person's past writing samples, then drafts new content that matches their established voice. The voice owner's prior work rides into the agent as task attachments, never as inline prompt text.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a PII sanitizer that strips confidential identifiers from source documents before any model call, and an `after-llm-response` guardrail that verifies the draft does not reproduce voice-owner phrases verbatim or surface any confidential tokens that slipped through.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the writing corpus lives in-process and the agent's tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.content-editorial.voice-ghostwriter  ~/my-projects/ghostwriter
cd ~/my-projects/ghostwriter
```

(Optional) Edit `SPEC.md` to point at a different writing corpus or content format (e.g., switch from blog posts to internal memos, LinkedIn posts, or technical release notes).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **GhostwriterAgent** — an AutonomousAgent that accepts a writing brief plus voice-sample attachments and returns a typed `DraftResult`.
- **DraftWorkflow** — orchestrates sanitize-wait → draft → guardrail-check per submitted brief.
- **DraftEntity** — an EventSourcedEntity holding the per-draft lifecycle.
- **CorpusSanitizer** — a Consumer that subscribes to `BriefSubmitted` events, redacts PII from all attached writing samples, and emits `CorpusSanitized` back to the entity.
- **DraftView + DraftEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded writing-corpus examples for your own (the JSONL file under `src/main/resources/sample-events/voice-samples.jsonl` after generation).
- `SPEC.md §5` — extend `DraftResult` with content-type-specific fields (e.g., `targetWordCount`, `publicationChannel`, `toneScore`).
- `prompts/ghostwriter.md` — narrow the agent's role (a marketing team would constrain it to brand-voice guidelines; an engineering team to changelog format rules).
- `eval-matrix.yaml` — wire a real PII redactor by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a writing brief + voice samples → the corpus is sanitized → a draft is produced that matches the voice owner's style markers → the draft appears in the UI.
2. The agent produces a draft that reproduces a verbatim phrase from the source corpus on first try → the `after-llm-response` guardrail rejects it → the agent retries → a clean draft lands.
3. PII strings present in submitted voice samples never appear in the LLM call log; only the redacted form does.
4. A draft whose content quality falls below the threshold receives a guardrail-rejection record in the UI with a clear reason.

## License

Apache 2.0.
