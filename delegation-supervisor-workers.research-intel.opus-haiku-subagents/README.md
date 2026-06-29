# Akka Sample: Haiku Sub-Agents under Opus (multimodal)

An Opus coordinator receives a multimodal analysis request — a batch of images plus a question — and delegates per-image analysis to a pool of Haiku sub-agents running in parallel. Once all sub-agents return, the coordinator synthesises their individual image reports into one unified response and runs an output guardrail before the answer reaches the user.

The blueprint demonstrates cost/latency-optimised multi-agent composition: expensive reasoning happens once in the coordinator; cheap, fast Haiku calls handle the repetitive per-item work.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: **Playwright** (for browser-driven image capture in the App UI integration test) **or** an [E2B](https://e2b.dev/) account with `E2B_API_KEY` set (for sandboxed image execution). The blueprint itself runs without either; they are needed only to exercise the image-injection test path in `reference/user-journeys.md`.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. The Opus model handles coordination; the Haiku model handles per-image analysis. If you have an Anthropic key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.research-intel.opus-haiku-subagents  ~/my-projects/haiku-subagents
cd ~/my-projects/haiku-subagents
```

(Optional) Edit `SPEC.md` to change the model names, the per-image prompt, or the PII redaction rules.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **OpusCoordinator** — AutonomousAgent (Opus model) that decomposes an image batch request into per-image tasks and later synthesises sub-agent reports into a final answer.
- **HaikuImageAnalyst** — AutonomousAgent (Haiku model) that analyses a single image and returns a typed `ImageReport`. One instance per image, run in parallel.
- **PiiSanitizer** — a component that redacts personally identifiable information from base64 image payloads before they are forwarded to any sub-agent.
- **AnalysisWorkflow** — Workflow that fans out to one `HaikuImageAnalyst` per image in parallel, joins their reports, and calls `OpusCoordinator` for synthesis.
- **AnalysisJobEntity** — EventSourcedEntity holding the full job lifecycle.
- **AnalysisView** — projection the UI streams via SSE.
- **AnalysisEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the number of images the simulator drips per batch, or disable the simulator.
- `SPEC.md §5` — adjust the `ImageReport` record fields (e.g., add `confidenceScore`).
- `prompts/opus-coordinator.md` — change how the coordinator frames the synthesis.
- `prompts/haiku-image-analyst.md` — adjust the per-image analysis instruction.
- `eval-matrix.yaml` — add a `before-tool-call` sanitizer if you wire a real image API.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit an image batch → job enters `QUEUED`, then `ANALYSING`, then `SYNTHESISED`.
2. One sub-agent fails → job enters `PARTIAL` with the reports that did succeed.
3. Guardrail rejects a synthesis that references PII it should not have seen → job enters `BLOCKED`.
4. Wait for the eval sampler → the job's row shows a quality score.

## License

Apache 2.0.
