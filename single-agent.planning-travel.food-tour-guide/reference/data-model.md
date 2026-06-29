# Data model — gemma-food-tour-guide

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TourPreferences` | `city` | `String` | no | Target city name (must match a seeded city profile). |
| | `durationDays` | `int` | no | Trip length in days (1–7). |
| | `dietaryStyles` | `List<String>` | no | User-selected dietary labels (e.g., `["vegetarian", "gluten-free"]`). May be empty if notes cover all requirements. |
| | `budgetTier` | `BudgetTier` | no | Enum value. |
| | `notes` | `String` | no | Free-text field; normalized by `PreferenceValidator` before LLM sees it. |
| | `requestedBy` | `String` | no | User identifier. |
| | `requestedAt` | `Instant` | no | When the endpoint received the request. |
| `ValidatedPreferences` | `city` | `String` | no | Unchanged from `TourPreferences.city`. |
| | `durationDays` | `int` | no | Unchanged. |
| | `normalizedCategories` | `List<String>` | no | Merged and deduplicated from `dietaryStyles` + notes-derived labels (e.g., "celiac" → "gluten-free"). |
| | `budgetTier` | `BudgetTier` | no | Unchanged. |
| `VenueStop` | `venueId` | `String` | no | Stable id from city profile, or `novel-<uuid>` if agent invented the venue. |
| | `venueName` | `String` | no | Human-readable venue name. |
| | `neighborhood` | `String` | no | City sub-district. |
| | `mealSlot` | `MealSlot` | no | Enum value. |
| | `dietaryCategory` | `String` | no | Primary dietary tag this stop satisfies. |
| | `culturalNote` | `String` | no | One sentence of cultural context. |
| | `description` | `String` | no | Two or more sentences; must exceed 20 characters. |
| `DayPlan` | `dayIndex` | `int` | no | 1-based; must be ≤ `durationDays`. |
| | `stops` | `List<VenueStop>` | no | 3–5 stops per day. |
| `TourItinerary` | `decision` | `ItineraryDecision` | no | Enum value. |
| | `summary` | `String` | no | 1–3 sentences. |
| | `days` | `List<DayPlan>` | no | One entry per requested `durationDays`. |
| | `generatedAt` | `Instant` | no | When the agent returned. |
| `CoverageResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `CoverageScorer` finished. |
| `TourRequest` (entity state) | `tourId` | `String` | no | — |
| | `preferences` | `Optional<TourPreferences>` | yes | Populated after `TourRequested`. |
| | `validated` | `Optional<ValidatedPreferences>` | yes | Populated after `PreferencesValidated`. |
| | `itinerary` | `Optional<TourItinerary>` | yes | Populated after `ItineraryRecorded`. |
| | `coverage` | `Optional<CoverageResult>` | yes | Populated after `CoverageScored`. |
| | `status` | `TourStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `TourRequested` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `TourRequest` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`BudgetTier`: `STREET_FOOD`, `MID_RANGE`, `FINE_DINING`.
`MealSlot`: `BREAKFAST`, `LUNCH`, `DINNER`, `SNACK`, `MARKET`.
`ItineraryDecision`: `FULL_PLAN`, `PARTIAL_PLAN`, `NEEDS_MORE_INFO`.
`TourStatus`: `REQUESTED`, `PREFERENCES_VALIDATED`, `GENERATING`, `ITINERARY_RECORDED`, `SCORED`, `FAILED`.

## Events (`TourRequestEntity`)

| Event | Payload | Transition |
|---|---|---|
| `TourRequested` | `preferences` | → REQUESTED |
| `PreferencesValidated` | `validated` | → PREFERENCES_VALIDATED |
| `GenerationStarted` | — | → GENERATING |
| `ItineraryRecorded` | `itinerary` | → ITINERARY_RECORDED |
| `CoverageScored` | `coverage` | → SCORED (terminal happy) |
| `TourFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `TourRequest.initial("")` with all `Optional` fields as `Optional.empty()` and `status = REQUESTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`TourRow` mirrors `TourRequest`. The view declares ONE query: `getAllTours: SELECT * AS tours FROM tour_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`TourTasks.java`)

```java
public final class TourTasks {
  public static final Task<TourItinerary> PLAN_FOOD_TOUR = Task
      .name("Plan food tour")
      .description("Read the city profile and traveler preferences and produce a TourItinerary covering every requested dietary style")
      .resultConformsTo(TourItinerary.class);

  private TourTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
