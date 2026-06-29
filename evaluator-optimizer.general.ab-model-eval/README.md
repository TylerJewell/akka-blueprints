# Akka Sample: A/B Testing Models

Two candidate models answer the same task prompt. A judge agent scores each response independently against a fixed rubric, picks the winner, and records the outcome as a typed eval event. Over time the recorded outcomes form a continuous accuracy signal that an operator uses to decide which model to promote.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **none**. Both model slots and the judge are modeled inside the service; no external router or gateway is required.

## Generate the system

```sh
cp -r ./evaluator-optimizer.general.ab-model-eval  ~/my-projects/ab-model-eval
cd ~/my-projects/ab-model-eval
```

(Optional) Edit `SPEC.md` to change the task prompt set, the judge rubric, or the recertification threshold.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CandidateAAgent** — AutonomousAgent that answers a task prompt as candidate model A.
- **CandidateBAgent** — AutonomousAgent that answers the same task prompt as candidate model B.
- **JudgeAgent** — AutonomousAgent that scores both responses against the rubric and emits a `JudgementRecord` with a winner and per-dimension scores.
- **EvalWorkflow** — Workflow that fans out to both candidates in parallel, collects results, calls the judge, and writes the outcome.
- **TrialEntity** — EventSourcedEntity that holds the trial lifecycle, both candidate responses, the judge's verdict, and the final winner.
- **TaskQueueEntity** — EventSourcedEntity that logs each submitted task prompt for replay and audit.
- **TrialsView** — read-side projection that the UI lists and streams via SSE.
- **TaskSubmissionConsumer** — Consumer that starts a workflow per inbound task.
- **TaskSimulator** — TimedAction that drips a sample task prompt every 60 s so the App UI is never empty.
- **AccuracySampler** — TimedAction that records per-trial eval events on a 30 s cycle (control E1).
- **EvalEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the task prompts the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Trial` record fields (e.g., raise `timeoutSeconds`, change the rubric dimensions).
- `prompts/candidate-a.md` — point candidate A at a different model family or persona.
- `prompts/candidate-b.md` — point candidate B at a different model family or persona.
- `prompts/judge.md` — change the rubric dimensions or scoring scale.
- `eval-matrix.yaml` — tighten enforcement on the accuracy eval (currently non-blocking advisory) or change the recertification gate threshold.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a task prompt → trial progresses `RUNNING` → `JUDGED` with both responses and a declared winner.
2. The periodic accuracy sampler records one `EvalRecorded` event per completed trial; the App UI timeline shows each verdict.
3. The recertification gate fires when the win-rate of the currently preferred model drops below the configured threshold.
4. A trial whose both candidates time out lands in `TIMED_OUT` with no winner declared and a structured failure reason.

## License

Apache 2.0.
