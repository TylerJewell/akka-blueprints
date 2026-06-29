# Akka Sample: Fair CV Matcher

Submit a CV and a list of job postings. One agent extracts a structured profile, protected-class signals are stripped, a second agent scores fit for each role, and a human reviewer monitors the results.

## Prerequisites

- Claude Code with the Akka plugin installed ([install docs](https://doc.akka.io/)).
- One model-provider API key, sourced at generation time. You will be asked how to provide it; options range from a mock provider (no key) to naming an existing environment variable. The key value is never written to disk.
- Host software: none. This blueprint runs out of the box — every external surface is modeled inside the same Akka service.

## Generate the system

1. Copy this folder into your own project location.
2. Optionally edit `SPEC.md` to change the system name, model provider, or the sample job postings.
3. In Claude Code, run:

   ```
   /akka:specify @SPEC.md
   ```

That single command scaffolds the specification and then auto-chains `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build`. When the service is up, Claude prints the listening URL.

## What you'll get

- `ExtractionAgent` — an AutonomousAgent that turns raw CV text into a structured profile.
- `MatchingAgent` — an AutonomousAgent that scores the sanitized profile against each job posting.
- `MatchingWorkflow` — a sequential workflow: extract → sanitize → match → review.
- `CandidateEntity` — an event-sourced record of each candidate's lifecycle.
- `CvIntakeQueue` + `CvIntakeConsumer` — inbound submissions starting a workflow each.
- `MatchesView` — the read model the UI lists and streams.
- `CvSimulator` — drips sample CVs every 30 seconds.
- `FairnessMonitor` — periodically checks selection-rate parity across demographic slices.
- `MatchingEndpoint` + `AppEndpoint` — the `/api` surface and the embedded UI.

## Customise before generating

- System name and short name — `SPEC.md` Section 1.
- Model provider and model name — `SPEC.md` Section 11 (Implementation directives).
- Sample CVs and job postings — referenced from `SPEC.md` Section 11; edit the companion `sample-data` lines.

## What gets validated

The user journeys in `reference/user-journeys.md`:

- A submitted CV is extracted, sanitized, and scored against every posting.
- Protected-class signals do not reach the matching agent.
- A reviewer records oversight on a matched candidate.
- The fairness monitor flags a slice whose selection rate drifts below parity.

## License

Apache 2.0. See `LICENSE`.
