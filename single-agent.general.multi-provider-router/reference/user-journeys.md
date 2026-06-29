# User journeys — multi-provider-router

## J1 — Submit a factual prompt and get a response

**Preconditions:** Service running on declared port (`http://localhost:9671/`); a valid model-provider API key set for at least one provider, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9671/` → App UI tab.
2. Click **Load seeded example** to fill the prompt textarea with the factual question seed ("What is the capital of Portugal?").
3. Leave the **Provider hint** on `auto`.
4. Click **Send**.

**Expected:**
- The new card appears in the live call list with status `PENDING` within 1 s.
- The card transitions to `DISPATCHED` within 1 s. The provider badge appears (OPENAI or ANTHROPIC).
- Within 30 s the card reaches `COMPLETED`. The right-pane detail shows: the provider badge, the response text, and the latency chip.
- The latency chip shows a non-zero value in milliseconds.

## J2 — Load balancing distributes calls across providers

**Preconditions:** Service running with mock LLM selected (`model-provider = mock`). The mock alternates Provider.OPENAI and Provider.ANTHROPIC on even/odd seeds.

**Steps:**
1. Submit 6 prompts using J1 steps repeated 6 times (use any mix of seed prompts or custom text).
2. Inspect the live call list after all 6 cards reach `COMPLETED`.

**Expected:**
- The live call list shows 6 cards.
- At least 2 cards have provider badge `OPENAI` and at least 2 have `ANTHROPIC`. Neither provider handled all 6 calls.
- No call ended in `FAILED` status (mock LLM is configured to succeed on all entries).

## J3 — Performance monitor produces eval report after 5 calls

**Preconditions:** Service running. Any provider (real or mock). Starting from a clean state (no prior calls) or after the previous window has closed.

**Steps:**
1. Submit 5 prompts in sequence (J1 × 5). Wait for each to reach `COMPLETED` before submitting the next.
2. Open the **Eval Matrix** tab.

**Expected:**
- The Live Monitor panel shows at least one `MonitorRow`. The window size is 5.
- The panel shows `ProviderStats` for each provider that handled calls in the window.
- P50 and P95 latency values are non-zero integers.
- The quality score chip is coloured (green, yellow, or red) based on the responses in that window.
- `GET /api/monitor` returns a JSON array with at least one entry.

## J4 — Provider hint pins a call to the specified backend

**Preconditions:** Service running with both providers configured (real keys or mock alternating mode).

**Steps:**
1. Set the **Provider hint** radio to `openai`.
2. Submit 2 prompts.
3. Set the **Provider hint** radio to `anthropic`.
4. Submit 2 prompts.

**Expected:**
- The first 2 cards have provider badge `OPENAI` regardless of the shuffle ordering.
- The next 2 cards have provider badge `ANTHROPIC`.
- All 4 calls reach `COMPLETED`.

## J5 — Failed call records error reason

**Preconditions:** Mock LLM mode. The mock includes at least one entry configured to fail (or the test is run with a deliberately invalid API key and a non-mock provider configuration).

**Steps:**
1. Configure the service to fail on a specific callId (or use an invalid key for one provider).
2. Submit a prompt that triggers the failure path.
3. Observe the live call list.

**Expected:**
- The card transitions from `DISPATCHED` to `FAILED`.
- The right pane shows an `errorReason` string describing why the call failed (e.g., "Provider timeout after 60s" or "Provider returned error 429").
- `GET /api/calls/{id}` returns the call row with `status: "FAILED"` and a non-null `errorReason`.
- No partial `result` is displayed; the result field is `null` in the JSON.
