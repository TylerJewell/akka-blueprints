# Akka Sample: Hello Agent

Ask a question. One agent answers it, with a confidence score.

The minimum-viable autonomous agent: a single `QuestionAnswerer` runs one ANSWER task and returns a typed `Answer`. No coordination, no tools — just one durable agent loop with an output check before the answer reaches the user.

## Prerequisites

- Claude Code with the Akka plugin installed. See the install docs: <https://doc.akka.io>.
- A model-provider API key — one of `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY`. If none is set, the generator offers a mock provider that needs no key.
- Host software: None. This blueprint runs out of the box.

## Generate the system

1. Copy this folder into your own project location.
2. Optionally edit `SPEC.md` — system name, model provider, the answer-quality threshold.
3. In Claude Code, run:

   ```
   /akka:specify @SPEC.md
   ```

That one command scaffolds the specification and auto-chains `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build`. When the service is up, Claude prints the listening URL.

## What you'll get

- `QuestionAnswerer` — an AutonomousAgent that answers a question and returns a typed `Answer{ text, confidence }`.
- `AnswererTasks` — the companion task definitions the agent accepts.
- `QuestionEntity` — an event-sourced record of each question's lifecycle (asked, answered, blocked).
- `QuestionsView` — the read model the UI lists and streams.
- `AnswerConsumer` — reacts to a new question and runs the answer task.
- `AskEndpoint` + `AppEndpoint` — the `/api` surface and the embedded UI.

## Customise before generating

- **System name / model provider** — `SPEC.md` Section 1 and Section 11.
- **Answer-quality threshold** — the before-agent-response check in `SPEC.md` Section 8 and `eval-matrix.yaml` control `G1`.
- **Agent behavior** — `prompts/question-answerer.md`.

## What gets validated

The user journeys in `reference/user-journeys.md`:

1. Ask a question via the UI and watch it move to `ANSWERED` with a confidence score.
2. Ask via the API and read the typed answer back.
3. Submit a question whose answer fails the output check and watch it land in `BLOCKED`.

## License

Apache 2.0. See `LICENSE`.
