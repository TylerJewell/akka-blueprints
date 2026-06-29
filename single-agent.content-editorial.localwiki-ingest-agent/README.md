# Akka Sample: LocalWiki Demo Agent

A single ingest agent accepts a URL, image, or PDF, fetches and parses the source, sanitizes PII out of the raw content, then files a structured markdown page into the wiki with one agent call. The file-write action is guarded before execution so path traversal and unsafe targets are caught before any bytes land on disk.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a PII sanitizer that runs before the agent ever sees the fetched content, and a `before-tool-call` guardrail that validates every proposed file-write against path safety rules and a blocklist of forbidden destinations.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The wiki corpus lives in-process; the agent's fetch and write operations are simulated by in-process stubs.

## Generate the system

```sh
cp -r ./single-agent.content-editorial.localwiki-ingest-agent  ~/my-projects/localwiki-ingest-agent
cd ~/my-projects/localwiki-ingest-agent
```

(Optional) Edit `SPEC.md` to point at a different wiki namespace, change the PII categories the sanitizer redacts, or extend the guardrail's blocklist of forbidden URL patterns.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **WikiIngestAgent** — an AutonomousAgent that receives a source URL/image/PDF as a task attachment and a set of page-filing instructions, then calls a `write_wiki_page` tool to file a structured markdown page in the wiki.
- **IngestWorkflow** — orchestrates sanitize-wait → ingest → index per submitted source.
- **IngestEntity** — an EventSourcedEntity holding the per-ingest lifecycle.
- **ContentSanitizer** — a Consumer that subscribes to `SourceSubmitted` events, redacts PII from the fetched content, and emits `ContentSanitized` back to the entity.
- **WikiPageView + IngestEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded URL, image, and PDF examples (`src/main/resources/sample-events/seed-sources.jsonl` after generation).
- `SPEC.md §5` — extend `WikiPage` with project-specific fields (e.g., `namespace`, `tags`, `expiresAt`).
- `prompts/wiki-ingest-agent.md` — narrow the agent's filing rules (a product-docs deployer would enforce a strict heading schema; an internal-knowledge deployer might add a summary-length limit).
- `eval-matrix.yaml` — wire a real path-safety library in the guardrail's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a URL → it is fetched and sanitized → the agent files a well-formed wiki page → the page appears in the UI.
2. The agent proposes a write to a forbidden path → the `before-tool-call` guardrail rejects it → the agent proposes an alternative path → the page lands in the correct location.
3. PII strings present in the fetched source never appear in the agent call; only the redacted form does.
4. A source that produces an empty or parse-failed body is recorded as FAILED with a clear reason; the entity preserves the partial state.

## License

Apache 2.0.
