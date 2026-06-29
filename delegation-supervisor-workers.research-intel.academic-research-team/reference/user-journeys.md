# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a query and watch parallel synthesis

**Preconditions:** service running on `http://localhost:9686/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a research query (e.g., "What are recent advances in LLM alignment?") and Submit.
2. Observe the new digest row via SSE.

**Expected:** the digest progresses `QUEUED → SCANNING → SYNTHESISED` within ~60 s. The expanded row shows a `PublicationBundle` (3–6 publications with titles, DOIs, and venues), a `TrendReport` (3–6 emerging areas and gaps), and a synthesised summary. Publications and trend analysis arrive close together because the workers ran in parallel.

## J2 — Worker timeout degrades the digest

**Preconditions:** `PaperScout` step timeout set to 1 s (test override).

**Steps:**
1. Submit a research query.
2. Watch the digest.

**Expected:** the `scoutStep` times out, the workflow routes to `degradeStep`, and the Coordinator synthesises from the DomainAnalyst output alone. The digest enters `DEGRADED`; the summary notes the missing publications side. No infinite retry.

## J3 — Citation guardrail blocks a digest with fabricated references

**Preconditions:** Coordinator returns a digest containing a DOI that is not present in the PaperScout's `PublicationBundle` (test fixture).

**Steps:**
1. Submit the fixture query.
2. Watch the digest.

**Expected:** `guardrailStep` flags the fabricated citation; the workflow calls `block`; the digest enters `BLOCKED` with a `failureReason` identifying which reference failed validation. The digest never appears as a finished result in the App UI's done state.

## J4 — Eval score appears beside a synthesised digest

**Preconditions:** at least one `SYNTHESISED` digest without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it directly via a test call.
2. Refresh the digest row in the App UI.

**Expected:** the digest gains an `evalScore` (1–5) and an `evalRationale`; the App UI row shows the score. Delivery was never blocked by the eval (non-blocking).

## J5 — Background load from the query simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running for at least 2 minutes.

**Expected:** `QuerySimulator` drips a query from `research-queries.jsonl` every 60 s; each becomes a digest that flows through the full pipeline. The App UI is non-empty on first load and gains new rows over time without any user input.

## J6 — Metadata tabs render from classpath

**Preconditions:** service running.

**Steps:**
1. Open the Risk Survey tab; open the Eval Matrix tab.

**Expected:** the Risk Survey tab renders the `risk-survey.yaml` contents with deployer-specific fields shown as muted italic "To be completed by deployer". The Eval Matrix tab shows both G1 and E1 rows with expandable rationale and implementation details.
