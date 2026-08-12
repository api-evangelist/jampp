---
name: Export a large Jampp report through an async pivot
description: Queue a large Jampp report with createAsyncPivot, poll asyncPivot until it reaches READY, and download the signed CSV result.
api: https://reporting-api.jampp.com/v1/graphql
operations:
  - Mutation.createAsyncPivot
  - Query.asyncPivot
  - Query.asyncPivots
generated: '2026-08-12'
method: generated
source: https://developers.jampp.com/docs/reporting-api/
---

# Export a large Jampp report through an async pivot

Use this when a synchronous `pivot` will not return — publisher-level (`siteBundle`)
reports, long periods, or any dimension combination Jampp refuses to run synchronously.
There is no pagination on the Reporting API; **asynchronous execution is the only
large-result mechanism**.

## 1. Authenticate

Same as the synchronous skill: `POST https://auth.jampp.com/v1/oauth/token` with
`grant_type=client_credentials`, form encoded, then
`Authorization: Bearer $TOKEN` on every call. Tokens live 7,200 seconds — long-running
exports can outlive one, so re-mint on 401 rather than assuming the job failed.

## 2. Queue the job

Unlike `pivot`, `createAsyncPivot` takes `dimensions` and `metrics` as **string arrays in
the input**, not as a selection set.

```graphql
mutation asyncSpendPerCampaign($from: DateTime!, $to: DateTime!) {
  createAsyncPivot(
    Input: {
      from: $from,
      to: $to,
      dimensions: ["campaignId", "campaign"],
      metrics: ["spend"]
    }
  ) {
    asyncPivot { id status }
  }
}
```

The response returns a UUID and an initial status, e.g.
`{"id": "3a818623-25f1-4b2b-ae07-67d09c173ed6", "status": "PENDING"}`.

**Record that id before doing anything else.** Jampp publishes no idempotency key and no
request-deduplication contract; calling `createAsyncPivot` again creates a *second* job
rather than returning the first. A retried mutation is a duplicate, not a no-op.

## 3. Poll

```graphql
query checkAsyncPivot($id: String!) {
  asyncPivot(pivotId: $id) { status url }
}
```

Use `asyncPivots(pivotIds: [...])` to check several at once, or with no argument to list
the caller's jobs — useful for recovering a lost id.

The state machine is `PENDING → QUEUED → RUNNING → READY`, plus two failure states:

* `UP_FOR_RETRY` — Jampp is retrying internally. Keep polling.
* `FAILED` — terminal. There is **no error message field** on `AsyncPivot`, so no
  diagnostic is returned. Re-issue with fewer dimensions or a shorter period; the
  documented causes are oversized output and forbidden dimension combinations.

No webhook or callback exists. Poll with your own backoff — Jampp documents none, and
returns no `Retry-After`.

## 4. Collect the result

On `READY`, `asyncPivot.url` holds a **signed URL** to a CSV of the result
(`outputType: S3`). Fetch it directly; no `Authorization` header is needed on the signed
URL, and it is a normal CSV with the requested dimensions and metrics as columns:

```
campaignId,campaign,spend
1,Campaign 1,10492.259863167761
```

Treat the URL as short-lived and secret — download once and store the CSV, do not pass the
URL around.

Related artifacts: `authentication/jampp-authentication.yml`,
`errors/jampp-problem-types.yml`, `conventions/jampp-conventions.yml`,
`data-model/jampp-data-model.yml`.
