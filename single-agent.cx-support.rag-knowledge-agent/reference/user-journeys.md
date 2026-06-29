# User journeys — rag-knowledge-agent

Acceptance journeys. Passing all of these means the blueprint generated correctly.

## J1 — Ask a question answerable from the document set

**Preconditions:** service running on `http://localhost:9443`; `DocLoader` has populated `DocIndexEntity`.
**Steps:**
1. Open the App UI tab; type a question covered by the bundled docs (e.g. about device setup).
2. Click **Ask**.
**Expected:** a session appears and moves `RECEIVED → RETRIEVED → ANSWERED`; the answer is non-empty and shows at least one citation referencing a bundled document. Within a moment it reaches `EVALUATED` with a faithfulness score.

## J2 — Ask an off-source question and get a refusal

**Preconditions:** as J1.
**Steps:**
1. Ask a question the documents do not cover (e.g. unrelated to the bundled topics).
2. Click **Ask**.
**Expected:** the session moves `RECEIVED → RETRIEVED → REFUSED`; `grounded=false`; a refusal reason is shown; no fabricated answer appears.

## J3 — Every answer carries a faithfulness score

**Preconditions:** at least one `ANSWERED` session from J1.
**Steps:**
1. Watch the session list over SSE after an answer is produced.
**Expected:** the `QueryAnswered` event triggers `FaithfulnessEvalConsumer`; the session reaches `EVALUATED` with `faithfulnessScore` in `[0, 1]` and a verdict of `supported`, `partial`, or `unsupported`.

## J4 — Document set loads at startup

**Preconditions:** service freshly started.
**Steps:**
1. Immediately after startup, ask a question covered by the docs.
**Expected:** retrieval returns passages (the index is non-empty because `DocLoader` ran), so the first question can be answered rather than refused for an empty index.

## J5 — Metadata tabs render

**Preconditions:** service running.
**Steps:**
1. Open the Eval Matrix tab, then the Risk Survey tab.
**Expected:** the Eval Matrix tab renders both controls (G1, E1) as `matrix-card` rows with colored mechanism pills; the Risk Survey tab renders pre-filled answers with `TO_BE_COMPLETED_BY_DEPLOYER` fields muted.
