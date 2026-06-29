# Akka Sample: GraphRAG Assistant

A single agent answers questions over an indexed document corpus, choosing local
(entity-neighborhood) or global (community-summary) retrieval per query, and
returning a grounded answer with citations.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One model-provider key, sourced any of five ways at generation time (real key never written to disk): `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider and need no key.
- Host software: none. This blueprint runs out of the box — the corpus index is built in-memory from bundled sample documents.

## Generate the system

```sh
cp -r ./single-agent.research-intel.graphrag-assistant  ~/my-projects/graphrag-assistant
cd ~/my-projects/graphrag-assistant
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or sample corpus.

In Claude Code, from inside the folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 makes Claude continue through
`/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` without further prompts.

## What you'll get

- `ResearchAgent` — one request/response Agent with two retrieval tools (local search, global search) and automatic scope selection.
- `CorpusIndex` — a KeyValueEntity holding the in-memory graph index (entities + community summaries) built from bundled docs.
- `QueryEntity` — an EventSourcedEntity tracking each question's lifecycle.
- `QueriesView` — a read model the UI queries and streams over SSE.
- `IndexBuilder` — a TimedAction that builds the index from sample documents at startup.
- `QueryEndpoint` + `AppEndpoint` — the JSON/SSE API and the single-file UI.
- A grounding guardrail and a PII sanitizer wired into the answer path.

## Customise before generating

- **System name / provider** — SPEC.md Section 1 and Section 11 identity block.
- **Sample corpus** — the `sample-docs/*.md` listed in SPEC.md Section 11 (Companion files).
- **Retrieval behavior** — `prompts/research-agent.md`.

## What gets validated

The user journeys in `reference/user-journeys.md`: ask a question that resolves
with local search, ask one that resolves with global search, watch a low-grounding
answer get blocked, and confirm corpus PII never reaches the answer.

## License

Apache 2.0.
