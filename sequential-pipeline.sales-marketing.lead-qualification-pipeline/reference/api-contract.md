# API contract — inbound-lead-qualification

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/leads` | `LeadFormData` | `201 { leadId }` | `LeadEndpoint` → `LeadEntity` |
| `GET` | `/api/leads` | — | `200 [ LeadRecord... ]` (newest-first) | `LeadEndpoint` ← `LeadView` |
| `GET` | `/api/leads/{id}` | — | `200 LeadRecord` / `404` | `LeadEndpoint` ← `LeadView` |
| `GET` | `/api/leads/sse` | — | `text/event-stream` | `LeadEndpoint` ← `LeadView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### LeadFormData (request body)

```json
{
  "firstName": "Jordan",
  "lastName": "Rivera",
  "email": "jordan.rivera@quantumsystems.io",
  "companyName": "Quantum Systems",
  "companySize": "XL",
  "message": "Interested in your compliance tooling for our platform team."
}
```

`companySize` enum values: `XS` (1–10), `S` (11–50), `M` (51–200), `L` (201–1 000), `XL` (1 001+).

### LeadRecord (response body — full)

```json
{
  "leadId": "lead-7f3a1b22",
  "formData": {
    "leadId": "lead-7f3a1b22",
    "firstName": "Jordan",
    "lastName": "Rivera",
    "email": "jordan.rivera@quantumsystems.io",
    "companyName": "Quantum Systems",
    "companySize": "XL",
    "message": "Interested in your compliance tooling for our platform team.",
    "submittedAt": "2026-06-28T10:00:00Z"
  },
  "profile": {
    "domain": "quantumsystems.io",
    "industry": "Enterprise Software",
    "estimatedArrBand": "$50M–$100M",
    "employeeBand": "1001+",
    "techStack": [
      { "technology": "Kubernetes", "evidence": "Listed in job postings" },
      { "technology": "Kafka", "evidence": "Engineering blog mentions" }
    ],
    "linkedinUrl": "https://linkedin.com/company/quantum-systems",
    "enrichedAt": "2026-06-28T10:00:05Z"
  },
  "score": {
    "leadId": "lead-7f3a1b22",
    "tier": "HOT",
    "score": 88,
    "rationale": "1 001+ employees and estimated ARR ≥ $50 M place this account in the HOT tier.",
    "assignedRepName": "Sarah Chen",
    "assignedRepSlackId": "U04XYZABC",
    "qualifiedAt": "2026-06-28T10:00:10Z"
  },
  "notification": {
    "channel": "C04SALESREP",
    "text": "HOT lead: Quantum Systems (1001+ employees)",
    "blocks": [
      { "type": "header", "text": "🔥 HOT lead: Quantum Systems" },
      { "type": "section", "text": "Score: 88/100 · Tier: HOT · Assigned to: Sarah Chen\nRationale: 1 001+ employees and estimated ARR ≥ $50 M." }
    ],
    "sentAt": "2026-06-28T10:00:15Z"
  },
  "eval": {
    "confidence": "HIGH",
    "rationale": "All three checks passed: tier-by-size, rep assignment, and score-tier alignment.",
    "evaluatedAt": "2026-06-28T10:00:15Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:15Z",
  "guardrailRejections": []
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). The `formData.email` and `formData.firstName` / `formData.lastName` fields ARE present on `GET /api/leads/{id}` — the PII sanitizer only operates on the agent's in-flight instruction context, not on the stored entity state. `guardrailRejections` is an array — empty on the happy path.

### LeadRecord (response body — mid-pipeline, QUALIFYING state)

```json
{
  "leadId": "lead-9c0e44d1",
  "formData": { "...": "..." },
  "profile": { "domain": "brightfutures.co", "...": "..." },
  "score": null,
  "notification": null,
  "eval": null,
  "status": "QUALIFYING",
  "createdAt": "2026-06-28T10:01:00Z",
  "finishedAt": null,
  "guardrailRejections": []
}
```

### Guardrail rejection in the response

```json
{
  "leadId": "lead-5af9c2dd",
  "guardrailRejections": [
    {
      "phase": "NOTIFY",
      "tool": "postLeadToSlack",
      "reason": "slack-write-blocked: postLeadToSlack requires status in {QUALIFIED, NOTIFYING} with score present, saw NOTIFYING with score absent",
      "rejectedAt": "2026-06-28T10:02:01Z"
    }
  ]
}
```

### SSE event format

```
event: lead-update
data: { "leadId": "lead-7f3a1b22", "status": "QUALIFIED", "score": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `ENRICHING`, `ENRICHED`, `QUALIFYING`, `QUALIFIED`, `NOTIFYING`, `NOTIFIED`, `EVALUATED`, `FAILED`) and one per `GuardrailRejected` audit event:

```
event: lead-rejection
data: { "leadId": "lead-7f3a1b22", "phase": "NOTIFY", "tool": "postLeadToSlack", "reason": "...", "rejectedAt": "..." }
```

Clients reconcile by `leadId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap `LeadEndpoint` with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend `LeadFormData` and `LeadSubmitted` to carry it. The PII sanitizer configuration should also be reviewed to ensure the pseudonymous tokens used in agent context cannot be reverse-mapped by an authenticated caller.
