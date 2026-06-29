# User journeys — citation-rag

## J1 — Submit a query and get a cited answer

**Preconditions:** Service running on declared port (`http://localhost:9234/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded query `What are the main findings on transformer attention efficiency?` has a matching `src/main/resources/sample-data/corpus/transformer-attention-efficiency.json` file.

**Steps:**
1. Open `http://localhost:9234/` → App UI tab.
2. From the **Pick a seeded query** dropdown, pick `What are the main findings on transformer attention efficiency?`.
3. Click **Run query**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `RETRIEVING` within 1 s more.
- Within ~20 s the card reaches `RETRIEVED`. The right pane shows the Retrieved-passages table with ≥ 4 rows; each row has a non-empty source, document title, and text preview.
- Within ~20 s more the card reaches `ATTRIBUTED`. The right pane shows ≥ 3 claims; every `claim.citedPassageId` resolves to a passage in the retrieved table.
- Within ~20 s more the card reaches `COMPOSED`, then `EVALUATED` within 1 s of that. The right pane shows an `Answer` with a response body, a claims list with citations, and the citation score chip showing 5/5.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — Citation guardrail blocks an uncited claim

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `compose-answer.json` includes one entry whose first `Claim` has `citedPassageId = null`.

**Steps:**
1. Submit the seeded query `What are the main findings on transformer attention efficiency?`.
2. Watch the query's lifecycle in the network panel of the browser dev tools (`/api/queries/sse`).

**Expected:**
- On the `composeStep`, the agent's first iteration returns an `Answer` containing a claim with `citedPassageId = null`. `CitationGuardrail` rejects the answer; a `CitationGuardrailRejected{claimId: "cl-bad0000", reason: "uncited-claim: ..."}` event lands on the entity.
- The rejected `Answer` is NEVER committed to the entity — no `AnswerComposed` event appears for the first iteration.
- The agent's second iteration produces a fully-cited answer. The card eventually reaches `EVALUATED` as in J1.
- The card in the App UI shows the small red dot indicating a rejection fired. The rejection-log strip on the right pane shows the one rejected claim with its full structured reason.

## J3 — Dangling citation flags eval score 1

**Preconditions:** Mock LLM mode. The mock's `compose-answer.json` includes one entry whose first `Claim` cites a `passageId` absent from the paired `PassageSet`.

**Steps:**
1. Submit any seeded query multiple times until the mock's dangling-citation entry is selected.
2. Watch the live list until a query card's border highlights red.

**Expected:**
- The flagged query's answer passed the citation guardrail (the `citedPassageId` was non-null) but the `passageId` does not resolve in the recorded `PassageSet`.
- The citation score chip shows **1** and the rationale reads: *"Resolved citations failed: claim 'cl-...' cites passageId 'p-...' which does not appear in the recorded PassageSet."*
- The card's border highlights red. The researcher sees the rationale inline and knows not to cite this answer without manual verification.
- The other queries in the run scored ≥ 4.

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so each tool call is logged. Any model provider.

**Steps:**
1. Submit any seeded query.
2. Wait for `EVALUATED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the queryId.

**Expected:**
- The RETRIEVE task's log entries show only `searchPassages` and `fetchPassage` calls.
- The ATTRIBUTE task's log entries show only `extractClaims` and `linkClaims` calls.
- The COMPOSE task's log entries show only `draftAnswer` and `formatCitations` calls.
- No cross-phase calls appear.
- The order of tasks in the log is RETRIEVE → ATTRIBUTE → COMPOSE. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J5 — Empty query with no matching corpus passages

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/corpus/<slug>.json` exists for the user's query.

**Steps:**
1. In the App UI, type a query for which no corpus entry exists (e.g. `history of ancient Roman plumbing`).
2. Click **Run query**.

**Expected:**
- `RetrieveTools.searchPassages` returns an empty list.
- The agent's RETRIEVE task returns a `PassageSet` with `passages = []`.
- The workflow advances to `attributeStep`. The ATTRIBUTE task returns a `ClaimSet` with `claims = []`.
- The workflow advances to `composeStep`. The COMPOSE task returns an `Answer` with `responseBody = "(no passages retrieved for this query)"`, empty `claims`, and empty `citations`. The citation guardrail accepts this (no claims to check).
- The citation score chip shows 1 (non-empty-answer check fails; all other rules are vacuously satisfied). The rationale names "no claims to evaluate."
- The pipeline completes; nothing crashes; the empty answer is honestly empty.

## J6 — Citation score reflects decoration citations

**Preconditions:** Mock LLM mode. A mock `compose-answer.json` entry whose `citations` list references a `passageId` from the `PassageSet` but no `Claim.citedPassageId` equals that `passageId`.

**Steps:**
1. Submit the seeded query `How do RAG systems handle conflicting sources?`.
2. Wait for `EVALUATED`.

**Expected:**
- The answer passed the citation guardrail (all claims have non-null, resolvable `citedPassageId` values).
- The `CoverageScorer` finds a passage in the `PassageSet` that is cited in the `citations` list but linked to no claim — a decoration citation.
- The citation score chip shows 4 (decoration-citation check fails; the other three checks pass). The rationale reads: *"Decoration citation: passage 'p-...' appears in citations but is not linked to any claim."*
- The researcher sees the 4/5 rating and can decide whether the decoration reference is intentional or a quality issue.
