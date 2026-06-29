# Akka Sample: Student Retention Outreach Agent

A continuous background worker monitors student engagement signals, applies FERPA-compliant PII redaction before any AI processing, classifies learners by attrition risk, and drafts advisor outreach messages that are held for human approval before sending. Demonstrates the **continuous-monitor** coordination pattern with two governance mechanisms: a PII sanitizer and a human-in-the-loop approval gate.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./continuous-monitor.education.retention-outreach  ~/my-projects/retention-outreach
cd ~/my-projects/retention-outreach
```

(Optional) Edit `SPEC.md` to point the `EngagementPoller` at a real LMS data source (Canvas, Moodle, Blackboard) or to keep the in-memory simulator.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **EngagementPoller** — TimedAction firing every 15 s that drips simulated student engagement events into `EngagementEventQueue`.
- **PiiSanitizer** — Consumer that redacts student identifiers (name, email, ID number, address) before any LLM call — satisfying FERPA's minimum-necessary principle.
- **AtRiskClassifierAgent** — Typed Agent that classifies each student signal as `AT_RISK`, `MONITOR`, or `ON_TRACK`.
- **OutreachDraftAgent** — AutonomousAgent that drafts an advisor outreach message for `AT_RISK` students.
- **StudentOutreachEntity** — EventSourcedEntity holding each student alert's lifecycle (signal received → sanitized → classified → drafted → approved/rejected → sent).
- **StudentOutreachWorkflow** — Workflow that orchestrates: sanitize → classify → (maybe draft) → wait for advisor approval → finalise.
- **OutreachView + OutreachEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **EvalRunner** — TimedAction running every 30 minutes; samples sent outreach messages and scores them for tone and policy adherence.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the simulated engagement feed for a real LMS webhook or polling integration.
- `SPEC.md §5` — add institution-specific fields (`cohort`, `financialAidStatus`, `advisorId`) to the engagement record.
- `prompts/at-risk-classifier.md` — tune the classification thresholds to your institution's at-risk definition.
- `eval-matrix.yaml` — wire a real PII redactor under the sanitizer mechanism's implementation.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A simulated engagement signal arrives → it is sanitized → classified → an outreach draft is produced → held in AWAITING_APPROVAL.
2. An advisor clicks Approve → the alert transitions to SENT (simulated; no real email leaves the process).
3. An advisor clicks Reject with a reason → the alert transitions to DISMISSED.
4. Eval scoring runs within 30 minutes and surfaces a score on sent messages.
5. PII from the raw engagement signal does not appear in the LLM call payload.

## License

Apache 2.0.
