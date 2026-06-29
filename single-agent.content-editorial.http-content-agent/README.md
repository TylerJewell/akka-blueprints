# Akka Sample: Claude AI Content Generator

An HTTP endpoint receives a content-generation prompt and returns a finished blog post or social
copy. One AI agent produces the draft; a `before-agent-response` guardrail scans the response for
brand-tone and safety issues before the text is returned to the caller. The output is stored on an
event-sourced entity so every generated piece is auditable.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: a
`before-agent-response` guardrail that checks the draft against a configurable brand-and-safety
ruleset before the response leaves the agent loop.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven
  Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have
  a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` /
  `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI,
  or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — content generation lives
  in-process and no external CMS connection is required.

## Generate the system

```sh
cp -r ./single-agent.content-editorial.http-content-agent  ~/my-projects/http-content-agent
cd ~/my-projects/http-content-agent
```

(Optional) Edit `SPEC.md` to adjust the brand ruleset (Section 8) or swap the seeded content
briefs (Section 3) for your own.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically
through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding
completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ContentGeneratorAgent** — an AutonomousAgent that accepts a content brief as a task
  instruction and returns a typed `GeneratedContent` record.
- **ContentWorkflow** — orchestrates the single generate step with a configurable retry budget.
- **ContentJobEntity** — an EventSourcedEntity holding the per-job lifecycle.
- **DraftGuardrail** — a `before-agent-response` hook that validates the draft against brand-tone
  and safety rules before it leaves the agent loop.
- **ContentView + ContentEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — replace the seeded content briefs with your own topics and target audiences.
- `SPEC.md §5` — extend `GeneratedContent` with channel-specific fields (e.g., `hashtags`,
  `callToActionUrl`, `wordCountTarget`).
- `prompts/content-generator.md` — narrow the agent's voice and style guide (a fintech deployer
  would constrain tone to formal/regulatory-compliant; a consumer brand to casual/energetic).
- `eval-matrix.yaml` — add additional guardrail checks (e.g., disallowed competitor mentions,
  required legal-disclaimer presence) by extending the `DraftGuardrail` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a content brief → the agent generates a draft → the guardrail passes it → the
   finished piece appears in the UI.
2. The agent returns a draft that fails the brand-safety check → the guardrail rejects it → the
   agent retries → the revised draft passes → the UI displays only the approved text.
3. A brief that produces a structurally incomplete draft (missing `title`, `body`, or `format`)
   is caught by the guardrail before it ever reaches the entity.
4. Multiple simultaneous submissions each maintain their own isolated job context; verdicts never
   bleed across jobs.

## License

Apache 2.0.
