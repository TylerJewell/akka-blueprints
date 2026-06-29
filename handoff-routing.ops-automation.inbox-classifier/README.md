# Akka Sample: InboxZero Lite — Email Classifier

A `ClassifierAgent` reads each inbound Gmail message, assigns it one of four labels (`URGENT`, `IMPORTANT`, `INFO`, `SPAM`), and hands the message off to a `RoutingAgent` that takes the configured action — flag for immediate attention, file in the right folder, mark as read, or move to spam. A PII sanitizer strips personal data from message bodies before any LLM call; a before-tool-call guardrail verifies that destructive actions (archive, delete, move-to-spam) are safe before they execute. Demonstrates the **handoff-routing** coordination pattern in an ops-automation context.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Integration-tier requirement: **a valid Gmail OAuth credential** (client ID + client secret + refresh token for the target inbox). Supply it using the same five key-sourcing options listed above — env var name, env file path, secrets-store URI, or type-once in this session. Akka never writes the credential value to disk.

## Generate the system

```sh
cp -r ./handoff-routing.ops-automation.inbox-classifier  ~/my-projects/inbox-classifier
cd ~/my-projects/inbox-classifier
```

(Optional) Edit `SPEC.md` to change the label taxonomy (add `NEWSLETTER`, `LEGAL`, etc.) or swap the simulated inbox source for the real Gmail adapter.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **InboxSimulator** — TimedAction firing every 30 s that drips canned email messages from a JSONL file into `InboxQueue`.
- **InboxQueue** — EventSourcedEntity append-only audit log of every inbound message (before redaction).
- **BodySanitizer** — Consumer that strips email addresses, phone numbers, national ID patterns, and account numbers from message bodies before any LLM call.
- **ClassifierAgent** — typed Agent that classifies the sanitized message into `URGENT`, `IMPORTANT`, `INFO`, or `SPAM` with a confidence and reason.
- **RoutingAgent** — AutonomousAgent that owns the `ROUTE` task for classified messages: decides the specific action (`FLAG_URGENT`, `MOVE_TO_FOLDER`, `MARK_READ`, `MOVE_TO_SPAM`, `FORWARD_TO_HUMAN`) and produces a typed `RoutingDecision`.
- **ActionGuardrail** — typed Agent used as a before-tool-call guardrail: checks any destructive action (`MOVE_TO_SPAM`, `DELETE`) before it executes. Returns `GuardrailVerdict { allowed, violations }`.
- **ClassificationJudge** — typed Agent that grades every classification decision against a 1–5 rubric.
- **ClassificationEvalScorer** — Consumer that listens for `MessageClassified` events and writes an inline eval score.
- **RoutingWorkflow** — Workflow per message: sanitize → classify → route → guardrail check → execute.
- **MessageEntity** — EventSourcedEntity holding each message's lifecycle.
- **MessageView + InboxEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the label taxonomy (add `NEWSLETTER`, `RECEIPT`, etc.) and add matching routing rules.
- `SPEC.md §5` — extend the `Message` record with deployer-specific fields (`threadId`, `listUnsubscribeHeader`, `organizationLabel`).
- `prompts/classifier-agent.md` — narrow the classification rules (urgency keywords, spam heuristics, domain-specific signal words).
- `prompts/routing-agent.md` — encode folder names, forwarding addresses, escalation thresholds.
- `eval-matrix.yaml` — swap the in-process PII regex for a real redactor or extend the guardrail rubric.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A simulated urgent message is sanitized, classified as `URGENT`, handed to `RoutingAgent`, and flagged for immediate attention.
2. A simulated spam message is classified as `SPAM` and the guardrail approves the move-to-spam action; the message is moved.
3. An ambiguous message is classified as `INFO` and filed in the general folder without triggering the guardrail.
4. A destructive action (delete) fails the before-tool-call guardrail and lands in `BLOCKED` for human review.
5. Every classified message carries a `ClassificationScore` (1–5) and rationale within a few seconds of the decision.

## License

Apache 2.0.
