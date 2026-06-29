# Akka Sample: Screenplay Writer

Paste a conversational thread (a newsgroup post chain or an email exchange) and the system turns it into a formatted screenplay: it redacts personal data from the source, extracts characters and setting, writes dialogue, then formats the scene.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY`. If none is set, `/akka:specify` offers a mock model that needs no key.
- Host software: none. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./sequential-pipeline.content-editorial.screenplay-writer  ~/my-projects/screenplay-writer
cd ~/my-projects/screenplay-writer
```

(Optional) Edit `SPEC.md` — system name, model provider, sample threads.

In Claude Code, from inside the folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding.

## What you'll get

- Three request/response agents: a thread analyzer, a dialogue writer, a screenplay formatter.
- A workflow that runs sanitize → analyze → write dialogue → format in order.
- An event-sourced entity holding each job's lifecycle and a view projecting it for the UI.
- A PII sanitizer that redacts names and addresses from source text before any agent sees it.
- A before-agent-response guardrail that checks the final screenplay for leaked personal data.
- An inbound thread queue, a consumer that starts one workflow per thread, and a timed simulator that drips canned threads.
- A self-contained five-tab UI served from the same service.

## Customise before generating

- `SPEC.md` Section 1 — system name and pitch.
- `SPEC.md` Section 11 — model provider and port.
- `reference/data-model.md` — the analysis fields the analyzer extracts.
- Sample threads referenced from Section 11 (`sample-events/threads.jsonl`).

## What gets validated

- Submitting a thread produces a completed screenplay with characters, dialogue, and scene formatting.
- Source names and email addresses are redacted before agents run; the redaction count is reported.
- A screenplay that would leak source PII is blocked by the final guardrail.
- The background simulator seeds threads and each becomes its own job without any UI interaction.

## License

Apache 2.0.
