# Architecture — web-scraper-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call and one outbound tool call. `ScrapeEndpoint` accepts a submission, writes a `ScrapeSubmitted` event onto `ScrapeEntity`, and immediately starts a `ScrapeWorkflow` instance. The workflow calls `ScrapeEntity.markFetching` to record that the scrape is in flight, then invokes `WebScraperAgent` — the single AutonomousAgent — with the target URL and extraction schema as task instructions.

Inside the agent's task loop, when the agent attempts to call `FetchPageTool`, the `UrlGuardrail` fires first via the `before-tool-call` hook. If the guardrail passes, `FetchPageTool` performs an HTTP GET and returns the raw HTML as a string. If it rejects, the agent receives a structured rejection as tool output, produces a BLOCKED `ScrapeResult`, and the workflow writes `ScrapeBlocked` — no outbound network call was made.

On a successful extraction, the workflow writes `RawContentExtracted` via `ScrapeEntity.recordResult`. The `ContentSanitizer` Consumer subscribes to that event, redacts PII from the summary and data points, and calls `ScrapeEntity.attachSanitized`. The workflow's `awaitSanitizedStep` polls the entity until the sanitized form arrives. `ScrapeView` projects every entity event into a read-model row; `ScrapeEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `ContentSanitizer` is a Consumer running deterministic regex logic — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct pause points:

1. The outbound HTTP fetch inside `FetchPageTool` — bounded by a 10-second socket timeout; in mock mode it reads from a bundled fixture file and returns immediately.
2. The `awaitSanitizedStep` polling loop inside `ScrapeWorkflow` — polls `ScrapeEntity` every 1 s up to its 15-second timeout, advancing as soon as `scrape.sanitized().isPresent()` returns true.

The agent call itself is bounded by `submitStep`'s 90-second timeout. The sanitizer Consumer runs in-process and finishes in milliseconds.

## State machine

Six states. The paths:

- The happy path is `SUBMITTED → FETCHING → EXTRACTED → SANITIZED`.
- Two failure paths branch from `FETCHING`: `BLOCKED` when the `UrlGuardrail` rejects the tool invocation, and `FAILED` when the agent call or the workflow step fails after exhausting its retry budget.
- `SUBMITTED → FAILED` covers the case where the workflow itself cannot start (rare runtime error).
- `BLOCKED` and `FAILED` are terminal states. A blocked scrape retains the submission data and the rejection reason on the entity, visible in the UI for the researcher to diagnose which policy was violated.
- There is no automatic re-queue or backoff for `BLOCKED` — the user must update the submitted URL or request an allowlist change.

## Entity model

`ScrapeEntity` is the source of truth. It emits six event types. `ScrapeView` projects every event into a row used by the UI. `ContentSanitizer` subscribes to entity events to compute the sanitized form. `ScrapeWorkflow` both reads (`getScrape`) and writes (`markFetching`, `recordResult`, `attachSanitized`, `block`, `fail`) on the entity. The relationship between `WebScraperAgent` and `ScrapeResult` is "returns" — the agent's task result is the extraction record.

## Governance flow

For any extraction that reaches `SANITIZED`:

1. **before-tool-call guardrail** — the `fetchPage` network call only fires after the domain is confirmed on the allowlist and the origin has capacity remaining in its rate-limit window.
2. **WebScraperAgent** — one model call, one structured output; the raw page HTML is the only input to the LLM.
3. **PII sanitizer** — the stored and displayed result is the redacted form; the raw extraction is audit-only on the entity.

Each step is independent. The guardrail does not know about PII; the sanitizer does not know about the allowlist. Removing one opens an explicit gap the other does not cover.
