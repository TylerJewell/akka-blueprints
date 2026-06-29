# Akka Sample: Lambda Error Diagnoser

A continuous background worker polls AWS Lambda error logs, diagnoses root causes with an AI agent, scores the diagnosis quality at the point of each incident, and surfaces actionable fix suggestions — with a full audit trail of every diagnosis decision. Demonstrates the **continuous-monitor** coordination pattern with one governance mechanism (on-incident eval).

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Integration-tier host software: **None**. The generated system ships with a built-in log simulator; no AWS account or CloudWatch access is required to run it.

## Generate the system

```sh
cp -r ./continuous-monitor.ops-automation.lambda-error-diagnoser  ~/my-projects/lambda-error-diagnoser
cd ~/my-projects/lambda-error-diagnoser
```

(Optional) Edit `SPEC.md` to point `LogPoller` at a real CloudWatch Logs source (or a Kinesis stream) instead of the in-memory simulator.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **LogPoller** — TimedAction firing every 20 s that drips simulated Lambda error events from a canned JSONL file.
- **LogNormalizer** — Consumer that extracts structured error context (function name, error type, stack trace excerpt, cold start flag) from raw log payloads before the AI sees anything.
- **DiagnosisAgent** — Agent (typed) that classifies the error into a root-cause category and recommends one concrete fix.
- **IncidentWorkflow** — Workflow that orchestrates: normalize → diagnose → eval → finalise per incident.
- **EvalJudge** — Agent (typed) that scores the diagnosis immediately after it is produced, before the result is surfaced to operators.
- **IncidentEntity** — EventSourcedEntity holding each incident's lifecycle (detected → normalized → diagnosed → evaluated → resolved/dismissed).
- **IncidentView + IncidentEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the simulated log source for a real CloudWatch Logs group or Kinesis data stream.
- `SPEC.md §5` — extend `LambdaLogEntry` with fields your functions already emit (`traceId`, `requestId`, `memoryUsedMB`).
- `prompts/diagnosis-agent.md` — scope the root-cause taxonomy to your specific runtime or tech stack.
- `eval-matrix.yaml` — promote `E1` to `enforcement: blocking` if you want diagnosis to hard-fail on low-confidence results before they surface.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A simulated error log arrives → it is normalised → diagnosed → evaluated → RESOLVED, all within 60 s.
2. An operator dismisses a false-positive incident from the UI.
3. The on-incident eval score appears on the incident card immediately after diagnosis.
4. A high-severity error triggers a diagnosis before any lower-severity backlog items.

## License

Apache 2.0.
