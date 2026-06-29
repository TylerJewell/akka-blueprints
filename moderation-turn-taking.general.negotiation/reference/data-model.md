# Data model

Every Java record, event, and enum the generated system defines. Lifecycle fields that are null until a later event fires are `Optional<T>` (Lesson 6); the offer history is a `List` that is empty, never null.

## `Negotiation` — entity state and View row

`domain/Negotiation.java`. This single record is both the `NegotiationEntity` state and the `NegotiationsView` row type.

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Negotiation id; equals the workflow id. |
| `item` | `Optional<String>` | until started | What is being negotiated. |
| `buyerBudget` | `Optional<Double>` | until started | Buyer's hard ceiling. |
| `sellerFloor` | `Optional<Double>` | until started | Seller's hard floor. |
| `status` | `NegotiationStatus` | no | Lifecycle stage. |
| `currentRound` | `int` | no | Current round, 0 before start, 1–10 while negotiating. |
| `offers` | `List<OfferLine>` | no (empty) | Append-only history of every turn. |
| `latestBuyerPrice` | `Optional<Double>` | until first buyer turn | Most recent buyer offer. |
| `latestSellerPrice` | `Optional<Double>` | until first seller turn | Most recent seller ask. |
| `outcome` | `Optional<String>` | until concluded | `"CONVERGED"` or `"NO_DEAL"`. |
| `finalPrice` | `Optional<Double>` | until converged | Settlement price; set only on a deal. |
| `finalTerms` | `Optional<String>` | until converged | One-line settlement terms; set only on a deal. |
| `startedAt` | `Optional<Instant>` | until started | When the negotiation began. |
| `concludedAt` | `Optional<Instant>` | until concluded | When the Facilitator concluded. |
| `escalatedAt` | `Optional<Instant>` | until escalated | When the stall monitor escalated it. |
| `outcomeScore` | `Optional<Double>` | until evaluated | Score from `OutcomeEvaluator`, 0.0–1.0. |
| `outcomeNotes` | `Optional<String>` | until evaluated | One-line explanation of the score. |

Helpers: `static Negotiation initial(String id)` returns `status = CREATED`, `currentRound = 0`, `offers = List.of()`, every `Optional` empty — and references no `commandContext()` (Lesson 3). `Negotiation applyEvent(NegotiationEvent e)` is a per-variant switch returning the next state.

## `OfferLine` — one turn in the history

`domain/OfferLine.java`.

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `round` | `int` | no | The round this offer belongs to. |
| `party` | `String` | no | `"BUYER"` or `"SELLER"`. |
| `price` | `double` | no | The offered price. |
| `terms` | `String` | no | The one-line clause attached. |
| `rationale` | `String` | no | One sentence explaining the move. |
| `accept` | `boolean` | no | Whether the party signalled willingness to settle. |
| `at` | `Instant` | no | When the turn was recorded. |

## `NegotiationStatus` — enum

```
CREATED · NEGOTIATING · CONCLUDED · ESCALATED
```

The deal-versus-no-deal split lives in the `outcome` string, not the enum, so no view query indexes the enum column (Lesson 2). `CONCLUDED` is reached on either a deal or a no deal.

## `NegotiationEvent` — sealed interface, six variants

`domain/NegotiationEvent.java`. Each variant carries `Instant timestamp()`.

| Event | Trigger | Carries |
|---|---|---|
| `NegotiationStarted` | `openStep` writes the opening state | `id, item, buyerBudget, sellerFloor, timestamp` |
| `OfferRecorded` | a buyer or seller turn completes | `id, round, party, price, terms, rationale, accept, timestamp` |
| `RoundAdvanced` | Facilitator returns `CONTINUE` | `id, newRound, timestamp` |
| `NegotiationConcluded` | `concludeStep` runs | `id, outcome, finalPrice (nullable), finalTerms (nullable), roundsUsed, timestamp` |
| `NegotiationEscalated` | `StalledNegotiationMonitor` fires | `id, timestamp` |
| `OutcomeEvaluated` | `OutcomeEvaluator` scores a concluded negotiation | `id, score, notes, timestamp` |

```java
sealed interface NegotiationEvent
    permits NegotiationStarted, OfferRecorded, RoundAdvanced,
            NegotiationConcluded, NegotiationEscalated, OutcomeEvaluated {
  Instant timestamp();
}
```

`NegotiationConcluded.finalPrice` and `.finalTerms` are nullable in the event (no settlement on a no-deal); the entity wraps them into the `Optional` row fields when applying.

## Agent result records

`application/Offer.java` and `application/FacilitatorDecision.java` — the typed results the agents return, retrieved via `forTask(taskId).result(...)`.

```java
record Offer(double price, String terms, String rationale, boolean accept) {}

record FacilitatorDecision(String verdict,      // "CONTINUE" | "CONVERGED" | "NO_DEAL"
                           double finalPrice,   // 0 unless verdict is CONVERGED
                           String finalTerms,   // "" unless verdict is CONVERGED
                           String reasoning) {}
```

## Request and queue records

```java
record StartRequest(String item, double buyerBudget, double sellerFloor) {}   // POST /api/negotiations body
record NegotiationRequestQueued(String item, double buyerBudget, double sellerFloor, Instant at) {}  // InboundRequestQueue event
```

## `NegotiationTasks` — companion to the AutonomousAgents (Lesson 7)

`application/NegotiationTasks.java` declares the three `Task<R>` constants the agents accept. Generating an AutonomousAgent without this file is a compile error.

```java
public final class NegotiationTasks {
  public static final Task<Offer> BUYER_OFFER = Task
    .name("Buyer offer").description("Produce the buyer's offer for the current round")
    .resultConformsTo(Offer.class);
  public static final Task<Offer> SELLER_OFFER = Task
    .name("Seller offer").description("Produce the seller's counteroffer for the current round")
    .resultConformsTo(Offer.class);
  public static final Task<FacilitatorDecision> FACILITATE = Task
    .name("Facilitate round").description("Decide continue, converged, or no-deal after both parties moved")
    .resultConformsTo(FacilitatorDecision.class);
}
```

## `SystemControl` state

`application/SystemControl.java` — a `KeyValueEntity`, single instance `"default"`, holding `record HaltState(boolean halted) {}`. Commands `halt`, `resume`, `isHalted`.

## View row note

`NegotiationsView` projects `NegotiationEntity` events into a row of type `Negotiation` (the same record above). The one query is `getAllNegotiations` → `SELECT * AS negotiations FROM negotiations_view`; a `streamAllNegotiations` variant feeds the SSE endpoint. No `WHERE status` clause — callers filter by status client-side (Lesson 2). Because the row type carries `Optional<T>` lifecycle fields, the view materializer accepts it (Lesson 6).
