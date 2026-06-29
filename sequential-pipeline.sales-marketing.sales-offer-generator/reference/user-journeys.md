# User journeys — sales-offer-generator

Acceptance journeys. Each generated system must pass all of them.

## J1 — Submit a brief and watch it reach APPROVED
- **Preconditions:** service running on `http://localhost:9341/`; a model provider configured (real or mock).
- **Steps:**
  1. Open the App UI tab.
  2. Paste a customer brief and click Submit.
  3. Watch the SSE list.
- **Expected:** the response returns an `offerId`. The offer appears in `QUEUED`, then advances `ANALYZED → MATCHED → PRICED → COMPOSED → APPROVED` (typically within ~30s). The expanded detail shows structured needs, at least one matched product, a pricing breakdown, and a non-empty offer document whose quoted total matches the pricing total.

## J2 — An over-ceiling discount is rejected
- **Preconditions:** a composed offer whose discount exceeds the policy ceiling or whose quoted total does not match `subtotal*(1-discountPct)` (reproducible with the policy-violating mock entry).
- **Steps:**
  1. Submit the brief that produces the violating offer.
  2. Watch the SSE list.
- **Expected:** the offer reaches `COMPOSED`, then transitions to `REJECTED`. The before-agent-response guardrail and `reviewStep` both flag the violation; the detail shows the policy reason. No `APPROVED` state is reached.

## J3 — Customer PII is redacted before any prompt
- **Preconditions:** a brief containing a synthetic email and phone number.
- **Steps:**
  1. Submit the brief.
  2. Inspect the stored offer (`GET /api/offers/{id}`) and the service logs.
- **Expected:** the stored `brief` and every persisted event carry redaction tokens in place of the email, phone, and full name; no log line emitted during the run contains the raw identifiers. The agents operate on the sanitized text only.

## J4 — Simulator seeds a brief with no interaction
- **Preconditions:** service running; no user action.
- **Steps:**
  1. Wait up to 30s.
- **Expected:** `RequestSimulator` reads the next line from `briefs.jsonl`, enqueues it, and a fresh `OfferWorkflow` runs the pipeline to a terminal state (`APPROVED` or `REJECTED`). A new offer appears in the SSE list without any submit.

## J5 — Catalog tool returns only canned products
- **Preconditions:** service running.
- **Steps:**
  1. Call `GET /api/catalog?segment=...` directly.
  2. Submit a brief and inspect the matched products.
- **Expected:** matched products' SKUs and list prices all come from the catalog response; no invented SKU or price appears in any offer.
