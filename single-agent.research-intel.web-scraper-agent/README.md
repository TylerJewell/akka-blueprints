# Akka Sample: Web Scraper Agent

A single research-intel agent receives a URL, fetches the page, and returns a structured extraction — title, summary, and key data points. Before the scraping tool fires, a `before-tool-call` guardrail validates the target URL against an allowlist and a per-origin rate-limit budget. After the page is fetched, a PII sanitizer redacts personal data from the extracted content before it is stored or forwarded downstream.

Demonstrates the **single-agent** coordination pattern in the research-intel domain, wired with two governance mechanisms: a `before-tool-call` guardrail that prevents scraping disallowed or rate-limited origins, and a PII sanitizer that runs on extracted content before it ever leaves the agent pipeline.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the HTTP fetch tool is a lightweight in-process call; seeded sample URLs resolve to static test fixtures bundled with the service.

## Generate the system

```sh
cp -r ./single-agent.research-intel.web-scraper-agent  ~/my-projects/web-scraper-agent
cd ~/my-projects/web-scraper-agent
```

(Optional) Edit `SPEC.md` to adjust the default domain allowlist in Section 3 (e.g., restrict to internal documentation hosts or expand to a curated set of public data sources).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **WebScraperAgent** — an AutonomousAgent that receives a scrape request (URL + extraction schema), calls the `fetchPage` tool to retrieve the raw HTML, and returns a typed `ScrapeResult`.
- **ScrapeWorkflow** — orchestrates guard-check → scrape → sanitize per submitted URL.
- **ScrapeEntity** — an EventSourcedEntity holding the per-scrape lifecycle.
- **ContentSanitizer** — a Consumer that subscribes to `RawContentExtracted` events, redacts PII from the extracted content, and emits `ContentSanitized` back to the entity.
- **ScrapeView + ScrapeEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap or extend the seeded domain allowlist (the JSONL file under `src/main/resources/sample-events/allowed-domains.jsonl` after generation).
- `SPEC.md §5` — add extraction-schema fields to `ScrapeResult` (e.g., `structuredData`, `languageCode`, `canonicalUrl`).
- `prompts/web-scraper.md` — narrow the agent's extraction instructions (a financial deployer would restrict it to press-release fields; a technical deployer to API documentation sections).
- `eval-matrix.yaml` — reference a real domain-allowlist service by naming it in the guardrail mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits an allowed URL → the guardrail passes → the page is fetched → the result is PII-sanitized → the structured extraction appears in the UI.
2. A user submits a blocked domain → the `before-tool-call` guardrail fires → the scrape is rejected without ever calling the fetch tool.
3. A page containing a person's email and phone number is scraped → the stored result shows only `[REDACTED-EMAIL]` and `[REDACTED-PHONE]`; the raw HTML is preserved on the entity for audit.
4. A user submits the same allowed origin five times in rapid succession → the rate-limit check in the guardrail blocks the fifth request until the window resets.

## License

Apache 2.0.
