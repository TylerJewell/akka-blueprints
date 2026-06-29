# Akka Sample: RAG Knowledge Agent

A user asks a question; the agent answers using only the supplied document set and cites the passages it used, or refuses when the documents do not support an answer.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One model-provider key, sourced any of five ways at generate time (mock provider needs none): `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY`.
- Host software: none. This blueprint runs out of the box — the document set and retrieval run in-process over files under `src/main/resources/sample-docs/`.

## Generate the system

```sh
cp -r ./single-agent.cx-support.rag-knowledge-agent  ~/my-projects/rag-knowledge-agent
cd ~/my-projects/rag-knowledge-agent
```

(Optional) Edit `SPEC.md`.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, then print the listening URL.

## What you'll get

- `KnowledgeAgent` (Agent) — grounded question answering with citations and a refusal path.
- `FaithfulnessAgent` (Agent) — per-answer faithfulness scoring against the retrieved passages.
- `DocIndexEntity` (KeyValueEntity) — in-memory chunk store with a cosine search command.
- `DocLoader` (TimedAction) — loads and embeds the sample documents into the index at startup.
- `QuerySessionEntity` (EventSourcedEntity) — the per-question lifecycle.
- `QuerySessionView` (View) — read model the UI queries and streams.
- `FaithfulnessEvalConsumer` (Consumer) — runs the faithfulness eval when an answer is produced.
- `KnowledgeEndpoint` + `AppEndpoint` (HttpEndpoint ×2) — REST + SSE surface and the static UI.

## Customise before generating

- System name and pitch — `SPEC.md` Section 1.
- Model provider — `SPEC.md` Section 11 (key sourcing) and `application.conf` after generation.
- Document set — replace the files under `src/main/resources/sample-docs/` with your own.

## What gets validated

The journeys in `reference/user-journeys.md`: a grounded question returns a cited answer; an off-source question is refused; each answer carries a faithfulness score; the document set loads at startup.

## License

Apache 2.0.
