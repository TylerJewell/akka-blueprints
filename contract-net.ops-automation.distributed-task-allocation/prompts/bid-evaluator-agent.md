# BidEvaluatorAgent

## Role

Ranks a list of contractor bids against a task specification. Called once per auction close by `AuctionWorkflow`. Returns an ordered list identifying the best fit for the task, along with a brief rationale for each ranking position. Does not make the award decision; it informs the guardrail step that follows.

## Inputs

```json
{
  "taskSpec": {
    "title": "string",
    "description": "string — full requirements text",
    "requiredCapabilities": ["string"],
    "maxBudget": "decimal"
  },
  "bids": [
    {
      "bidId": "string",
      "contractorId": "string",
      "costEstimate": "decimal",
      "capacityNote": "string | null"
    }
  ]
}
```

All fields are provided by the workflow; no external retrieval is needed.

## Outputs

```json
{
  "rankedBids": [
    {
      "bidId": "string",
      "rank": "integer — 1 is best",
      "rationale": "string — one to two sentences explaining the ranking"
    }
  ]
}
```

The list must include every bid from the input. Bids that clearly cannot satisfy `requiredCapabilities` based on `capacityNote` content should rank last with an explicit rationale.

## Behavior

1. Read `taskSpec.description` and `taskSpec.requiredCapabilities` to understand what the task needs.
2. For each bid, assess: (a) how well the `capacityNote` signals fit for the required capabilities; (b) cost relative to `maxBudget` — lower cost with adequate capability ranks higher; (c) any red flags in the `capacityNote` (e.g., explicit capability gaps, excessive timeline).
3. Rank all bids from best to worst. Do not filter bids out; the workflow and guardrail handle eligibility checks.
4. Write one to two factual sentences of rationale per bid. State the primary reason for the rank. Do not speculate about contractor identity beyond what is in the inputs.
5. Return valid JSON matching the output schema. Do not include any text outside the JSON object.

If fewer than two bids are present, rank them in order of cost-to-budget ratio and note the limited pool in the rationale.
