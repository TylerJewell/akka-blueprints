# User journeys — Meeting Prep Briefer

Acceptance tests as numbered journeys. Each passes when its expected state and UI updates hold.

## J1 — Submit a meeting request and watch it complete

**Preconditions:** service running on `http://localhost:9382`; one model provider configured (or mock).
**Steps:**
1. `POST /api/briefings` with `{ "meetingTopic": "Q3 renewal with Northwind", "participants": ["Dana Reyes", "Sam Okafor"] }`. Note the returned `briefingId`.
2. Subscribe to `/api/briefings/sse`.
**Expected:** the briefing appears and moves RECEIVED → PLANNING → RESEARCHING → COMPOSING → READY within ~60s. At READY, `briefingDoc` is non-empty and `readyAt` is set.

## J2 — Briefing carries all three sections

**Preconditions:** a briefing has reached READY (J1).
**Steps:**
1. `GET /api/briefings/{briefingId}`.
**Expected:** `talkingPoints` has 3-5 entries, `questions` has 3-5 entries, and `profiles` has one entry per submitted participant, each with a non-empty `redactedBackground`. The App UI tab renders all three sections when the row is expanded.

## J3 — Participant PII is redacted

**Preconditions:** the canned research source returns email/phone/address for a participant.
**Steps:**
1. Submit a request naming a participant whose `participants.json` entry contains contact details.
2. After READY, read `profiles[].redactedBackground`.
**Expected:** no email address, phone number, or street-address token appears in any stored `redactedBackground`, even though the raw research payload contained them. The S1 sanitizer stripped them before `ProfileAdded` was emitted.

## J4 — Out-of-scope research is blocked

**Preconditions:** service running.
**Steps:**
1. Submit a request. During research, the supervisor or a worker attempts a `ResearchSource` lookup for a name not in the participant list (simulated by the guardrail test path).
**Expected:** the G1 before-tool-call guardrail blocks the lookup; the worker returns an empty profile for that subject rather than retrying; the briefing still reaches READY using only in-scope research.

## J5 — Background load from the simulator

**Preconditions:** service running, no UI interaction.
**Steps:**
1. Wait ~30s.
**Expected:** `RequestSimulator` enqueues a canned request from `meeting-requests.jsonl`; `BriefingRequestConsumer` starts a workflow; a new briefing appears over SSE and runs to READY on its own.

## J6 — A failed worker fails the briefing cleanly

**Preconditions:** service running; a research step is forced to error (e.g. empty plan).
**Steps:**
1. Submit a malformed or unresearched request that drives `researchStep` into its recovery path.
**Expected:** the briefing transitions to FAILED with a recorded `failureReason`; no partial briefing is published; the UI shows the FAILED chip.
