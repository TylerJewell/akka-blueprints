# Akka Sample: Slack Lead Qualifier

A continuous background worker listens for Slack new-member events, enriches each lead with web-search data, scores the fit against configurable criteria, sanitizes PII from the enriched profile, and posts a qualification summary back to a designated Slack channel. Demonstrates the **continuous-monitor** coordination pattern wired with a before-tool-call guardrail and a PII sanitizer.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- A valid Slack credential (Bot Token with `team:read`, `users:read`, and `chat:write` scopes, **or** keep the in-process Slack simulator if you want no external dependency). Supply the token via the same key-sourcing mechanisms above — never written to disk.

## Generate the system

```sh
cp -r ./continuous-monitor.sales-marketing.slack-lead-qualifier  ~/my-projects/slack-lead-qualifier
cd ~/my-projects/slack-lead-qualifier
```

(Optional) Edit `SPEC.md` to point the Slack consumer at a real workspace or to keep the in-memory event simulator.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **SlackEventPoller** — TimedAction firing every 20 s that pulls simulated `member_joined_channel` events.
- **LeadEnrichmentAgent** — Agent that runs a web search for company and role context on the new member.
- **LeadScoringAgent** — Agent that produces a numeric fit score plus a qualification summary.
- **PiiSanitizer** — Consumer that strips PII from enriched lead data before the scoring LLM sees it.
- **SlackPostGuardrail** — before-tool-call hook that validates the Slack post for tone and residual PII before the call fires.
- **LeadEntity** — EventSourcedEntity holding each lead's lifecycle (received → enriched → sanitized → scored → posted / suppressed).
- **LeadWorkflow** — Workflow that orchestrates enrich → sanitize → score → post per lead.
- **LeadView + LeadEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the simulated Slack event source for a real Slack Events API webhook receiver.
- `SPEC.md §5` — extend the `EnrichedLead` record with org-specific scoring criteria (`targetIndustry`, `minHeadcount`, etc.).
- `prompts/lead-enrichment.md` — tune the web-search strategy for your ICP.
- `prompts/lead-scoring.md` — adjust the fit-score rubric and posting threshold.
- `eval-matrix.yaml` — add regulation anchors once you know which jurisdictions the deployer covers.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A simulated Slack member event arrives → enriched → sanitized → scored → posted.
2. A low-score lead is suppressed — no Slack post is made.
3. The PII-sanitized profile is what reaches the scoring LLM (audit check).
4. A Slack post containing residual PII is blocked by the guardrail.

## License

Apache 2.0.
