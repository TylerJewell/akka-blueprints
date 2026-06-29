# Akka Sample: Conditional Email/Interview Workflow

An `EmailClassifierAgent` reads an inbound recruiting email and routes it to the path that matches its content: `InfoRequester` for general questions, `InterviewOrganizer` (with RAG lookup and a scheduling tool) for interview coordination requests, or `RejectRouter` for non-qualifying candidates. A before-tool-call guardrail checks every scheduling or email-send operation before it fires, and a PII sanitizer strips candidate contact details before any LLM call.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host-software requirement: **None** — this blueprint runs out of the box. Inbound candidate emails and the recruiting calendar are simulated inside the service.

## Generate the system

```sh
cp -r ./handoff-routing.hr-recruiting.conditional-recruiting-router  ~/my-projects/conditional-recruiting-router
cd ~/my-projects/conditional-recruiting-router
```

(Optional) Edit `SPEC.md §3` to connect to a real email inbox, replace the simulated calendar with a calendar API call, or add routing paths (e.g. `ASSESSMENT_SENDER`).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **EmailSimulator** — TimedAction firing every 30 s that drips canned candidate emails from a JSONL file into `InboxQueue`.
- **InboxQueue** — EventSourcedEntity append-only log of every inbound email (audit before redaction).
- **CandidateSanitizer** — Consumer that redacts email addresses, phone numbers, and home addresses before any LLM call.
- **EmailClassifierAgent** — typed Agent that classifies the sanitized email into `INFO_REQUEST`, `INTERVIEW_REQUEST`, or `UNROUTABLE`.
- **InfoRequester** — AutonomousAgent that owns the `REPLY` task for information-request emails. Composes a factual answer from the job-posting context.
- **InterviewOrganizer** — AutonomousAgent that owns the `SCHEDULE` task for interview-request emails. Uses a RAG lookup for interviewer availability and a scheduling tool to emit a calendar hold.
- **RoutingJudge** — typed Agent used by `RoutingEvalScorer` to grade every routing decision against a 1–5 rubric.
- **ToolCallGuardrail** — typed Agent implementing the before-tool-call guardrail for scheduling and email-send operations.
- **RecruitingWorkflow** — Workflow per application: sanitize → classify → route → {reply | schedule} → guardrail → send.
- **ApplicationEntity** — EventSourcedEntity holding each application's lifecycle.
- **ApplicationView + RecruitingEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **RoutingEvalScorer** — Consumer that listens for `RoutingDecided` events and writes an inline eval score.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — add routing paths (`ASSESSMENT_SENDER`, `OFFER_COORDINATOR`) and wire the matching agent.
- `SPEC.md §5` — extend `Application` with deployer-specific fields (`requisitionId`, `hiringManagerId`, `slaDeadline`).
- `prompts/email-classifier-agent.md` — tighten the routing rules (minimum-qualification threshold, language-detection for multi-locale pipelines).
- `prompts/interview-organizer.md` — encode your actual interview-format catalogue (phone screen, technical, panel, executive), interviewer pool IDs, and calendar-block duration rules.
- `eval-matrix.yaml` — replace the in-process PII regex with a real redactor when deploying to production.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Simulator drips an info-request email → sanitized, classified `INFO_REQUEST`, handled by `InfoRequester`, reply sent.
2. Simulator drips an interview-request email → sanitized, classified `INTERVIEW_REQUEST`, `InterviewOrganizer` retrieves availability and emits a calendar hold, reply sent.
3. An unroutable email (off-topic or below threshold) is classified `UNROUTABLE` and the workflow terminates in `UNROUTABLE_CLOSED` without invoking any specialist.
4. A scheduling tool call with a missing interviewer slot is blocked by the before-tool-call guardrail; the application lands in `TOOL_BLOCKED` for recruiter review.
5. The routing eval score (1–5) appears on every classified application within ~10 s of the routing decision.

## License

Apache 2.0.
