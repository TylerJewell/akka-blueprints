# User journeys — personal-finance-agent

Numbered acceptance journeys. Each defines preconditions, steps, and the expected outcome the generated system must produce.

---

## J1 — Spending summary: read-only query, happy path

**Preconditions**

- The service is running and the `individual-checking` account set is seeded (2 accounts, 20 transactions spanning May–June 2026, at least 9 tagged Groceries and 5 tagged Dining).
- No write operations are involved.

**Steps**

1. Open the App UI tab.
2. Select account set `individual-checking`.
3. Enter query: "How much did I spend on groceries and dining last month?"
4. Enter principal: "user-0001".
5. Click **Ask**.

**Expected**

- A `queryId` is returned immediately.
- Within ~1 s the card transitions to `SANITIZED`; the PII-token count badge shows ≥ 1.
- Within ~30 s the card transitions to `ANSWERED`.
- The agent's answer paragraph references the total amount and number of transactions.
- `structuredData` is populated with a JSON array containing at least entries for `Groceries` and `Dining`, each with a positive `total` and `count ≥ 1`.
- The tool trace shows `listTransactions` (outcome `NOT_A_WRITE`) followed by `groupByCategory` (outcome `NOT_A_WRITE`). No write tools appear.
- The raw transaction data is not visible in the UI; only the sanitized form (with `[ACCT-...]` tokens) appears.

---

## J2 — Fund transfer: write approved by guardrail

**Preconditions**

- `individual-checking` account set is seeded; `acct-001` (Main Checking) has an available balance of $1,240.00; `acct-002` (Savings) is the only other account in the set.
- The principal `user-0001` matches the session account set owner.

**Steps**

1. Select account set `individual-checking`.
2. Enter query: "Transfer $100 to my savings account."
3. Enter principal: "user-0001".
4. Click **Ask**.

**Expected**

- Within ~30 s the card reaches `ANSWERED`.
- The agent's answer confirms the transfer was completed (e.g., "I've transferred $100.00 to your Savings account.").
- The tool trace shows `getBalance` (outcome `NOT_A_WRITE`) then `transferFunds` (outcome `APPROVED`).
- The write-outcome callout shows APPROVED in green.
- `structuredData` is `null` (no table needed for a simple confirmation).

---

## J3 — Fund transfer: blocked by guardrail (over-balance)

**Preconditions**

- Same `individual-checking` account set; `acct-001` has $1,240.00 available.
- The user requests a transfer amount that exceeds the balance.

**Steps**

1. Select account set `individual-checking`.
2. Enter query: "Transfer $5,000 to my savings account."
3. Enter principal: "user-0001".
4. Click **Ask**.

**Expected**

- Within ~30 s the card reaches `ANSWERED`.
- The agent's answer explains the block: it references the available balance and states that no funds were moved.
- The tool trace shows `getBalance` (outcome `NOT_A_WRITE`) then `transferFunds` (outcome `BLOCKED`). The blocked entry's result field contains a human-readable reason naming the balance limit.
- The write-outcome callout shows BLOCKED in red with the reason text.
- The simulated account balance is unchanged.
- No `FAILED` status — the query completed normally; the guardrail block is a valid answer, not an error.

---

## J4 — PII sanitisation: raw identifiers never reach the model

**Preconditions**

- The seeded `individual-checking` transactions include at least one description containing a full-name string matching the pattern for the principal and one account-number fragment of the form `4111-1111-xxxx-1234`.

**Steps**

1. Select account set `individual-checking`.
2. Enter query: "List my last 5 transactions."
3. Enter principal: "user-0001".
4. Click **Ask**.

**Expected**

- The sanitized context block in the UI shows `[NAME-INITIALS]` and `[ACCT-1234]` tokens — never the raw name or full account number.
- The LLM call log (accessible via the tool trace) contains only the sanitized descriptions; no raw PII appears.
- The entity's `request.rawTransactions` (accessible via `GET /api/queries/{id}`) still contains the original unredacted strings.
- The `piiTokensReplaced` count on the `SANITIZED` card badge is ≥ 2.

---

## J5 — Cross-account transfer attempt: blocked by guardrail (destination not owned)

**Preconditions**

- `individual-checking` account set is seeded with 2 accounts (`acct-001`, `acct-002`) belonging to `user-0001`.
- The user attempts to transfer to an account id not in the set.

**Steps**

1. Select account set `individual-checking`.
2. Enter query: "Transfer $50 to account acct-999."
3. Enter principal: "user-0001".
4. Click **Ask**.

**Expected**

- The card reaches `ANSWERED`.
- The tool trace shows `transferFunds` with outcome `BLOCKED`. The result field names the rule: destination account `acct-999` is not in the session's account list.
- The agent's answer explains the rejection clearly and does not attempt a retry.
- The write-outcome callout shows BLOCKED in red.

---

## J6 — Failed query: sanitizer error on malformed input

**Preconditions**

- The service is running. The client sends a `POST /api/queries` request where the resolved transaction set contains a record with a `null` `merchant` field, triggering a sanitizer null-pointer path.

**Steps**

1. Submit a query against account set `individual-checking` with principal `user-malform`.
2. The sanitizer encounters the null field during its pipeline.

**Expected**

- The card transitions to `SUBMITTED` then to `FAILED` within ~15 s (the `awaitSanitizedStep` timeout fires, or the Consumer emits a `QueryFailed` event directly).
- The UI shows the `FAILED` status pill in red.
- The entity's state shows `status = FAILED` and a non-empty `reason` string on the `QueryFailed` event.
- The partial data available before the failure (the raw query text and account list) is still readable via `GET /api/queries/{id}`.
- No agent call is made; the LLM call log is empty for this query.
