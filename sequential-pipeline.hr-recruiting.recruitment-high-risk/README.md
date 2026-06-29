# Akka Sample: Recruitment Pipeline

A sequential pipeline that sources candidate profiles, screens them against a job requisition, and scores the match — with a sourcing guardrail, special-category redaction, an operator halt switch, and deployer monitoring wired in.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY`. If none is set, `/akka:specify` offers a mock-LLM path that needs no key.
- Host software: none. This blueprint runs out of the box — the candidate source is modeled inside the service from canned data.

## Generate the system

```sh
cp -r ./sequential-pipeline.hr-recruiting.recruitment-high-risk  ~/my-projects/recruitment-pipeline
cd ~/my-projects/recruitment-pipeline
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or the role requirements.

In Claude Code, from inside the folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build`.

## What you'll get

- Three pipeline agents: `SourcingAgent`, `ScreeningAgent`, `MatchingAgent`.
- A `RecruitmentWorkflow` running source → sanitize → screen → match in sequence.
- A `CandidateEntity` (event-sourced) and a `CandidatesView` read model streamed over SSE.
- A `SystemControlEntity` halt switch the operator or a regulator can flip.
- A `RequisitionSimulator` that drips canned job requisitions, a `RequisitionConsumer` that starts one workflow per requisition, and a `DeployerMonitor` that aggregates outcome metrics for human-on-loop oversight.
- Two HTTP endpoints and a single self-contained UI with five tabs.

## Customise before generating

- System name and pitch — `SPEC.md` Section 1.
- Model provider — `SPEC.md` Section 11 (Generation workflow).
- Role requirements and the sourcing allowlist — `SPEC.md` Sections 5 and 8, `prompts/`.

## What gets validated

- A requisition produces a candidate that moves SOURCED → SANITIZED → SCREENED → MATCHED with a score.
- A sourcing call against a non-allowlisted domain is blocked before it runs.
- Protected attributes are redacted before any screening or scoring.
- An operator halt stops new evaluations; resume releases them.

See `reference/user-journeys.md`.

## License

Apache 2.0.
