---
name: Pull Jampp spend and funnel metrics by campaign
description: Authenticate against Jampp with OAuth 2.0 client credentials and run a synchronous pivot report that returns spend, impressions, clicks, installs and events grouped by campaign for a date range.
api: https://reporting-api.jampp.com/v1/graphql
operations:
  - Query.pivot
generated: '2026-08-12'
method: generated
source: https://developers.jampp.com/docs/reporting-api/
---

# Pull Jampp spend and funnel metrics by campaign

Use this for any question of the form "what did we spend, and what did it produce, per
campaign / country / app / publisher, between two dates". It is the Jampp reporting
happy path.

## 1. Get a token

The Reporting API is OAuth 2.0 client credentials. Credentials are minted by a human in the
Silver dashboard at `https://app.jampp.com/users/credentials` — you cannot self-register.

```
POST https://auth.jampp.com/v1/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id=$CLIENT_ID&client_secret=$CLIENT_SECRET
```

The response is `{"access_token": ..., "token_type": "Bearer", "expires_in": 7200}`.
The token lasts **two hours** and there is no refresh token — when it expires, ask again.

Two failure modes worth pre-empting:

* Sending JSON to the token endpoint returns `400 {"error":{"message":"Invalid request:
  content must be application/x-www-form-urlencoded"}}`. It must be form encoded.
* Calling the GraphQL endpoint with no `Authorization` header returns
  `401 {"error":"Neither Cookie nor Authorization present."}`.

## 2. Call the endpoint

`POST https://reporting-api.jampp.com/v1/graphql` with
`Authorization: Bearer $TOKEN` and `Content-Type: application/json`.

Ignore the "API Endpoints" block in Jampp's own documentation, which prints
`https://reporting-api.com/graphql` — that is not a Jampp host. Every worked example in
the same page, and the only host that answers, is `reporting-api.jampp.com`.

## 3. Run the pivot

`pivot` takes the period and the shaping arguments; the **dimensions and metrics are the
selection set**, not arguments.

```graphql
query spendPerCampaign($from: DateTime!, $to: DateTime!) {
  spendPerCampaign: pivot(from: $from, to: $to) {
    results {
      campaignId
      campaign
      spend
      impressions
      clicks
      installs
      events
    }
  }
}
```

Shape it with:

* `filter:` — field predicates, e.g. `filter: {campaignId: {equals: 1234}}`.
* `cleanup:` — predicates applied *after* the report is computed, e.g.
  `cleanup: {clicks: {greaterThan: "0"}}`. Operators: `greaterThan`,
  `greaterOrEqualThan`, `lowerThan`, `lowerOrEqualThan`.
* `options: {removeZeroRows: true}` — drop empty rows.
* `context: {sqlTimeZone: "America/Buenos_Aires"}` — a JodaTime zone applied to both the
  results and any input date without an explicit offset.
* `date(granularity: DAILY)` on the `date` dimension for a time series.
* `totals { ... }` alongside `results { ... }` for aggregates. `totals` accepts fewer
  dimensions, because not everything aggregates.

Reuse metric selections with fragments on `PivotMetrics`:

```graphql
fragment metrics on PivotMetrics { impressions clicks spend }
```

## 4. Respect the documented limits

These are structural, not a request budget — Jampp publishes no rate limit at all.

* `HOURLY` granularity exists only for the **last 15 days**; older data is rolled up to daily.
* `siteBundle` exists only for the **last 3 months**; beyond that it is NULLed.
* `siteName` and `siteBundle` **cannot** be queried together with `region`, `city`, `dma`,
  `zipCode`, `ad` or `adId`.
* Some combinations (notably `siteBundle` plus ad-level dimensions) will not run
  synchronously at all. If a query hangs or returns nothing, switch to the async skill.

## 5. Errors

There is no error catalog and no `extensions.code`. GraphQL errors come back as a plain
`errors[]` array of messages; transport errors come back as `{"error": ...}` — a string on
`reporting-api`, an object on `auth`. Branch on the HTTP status and the message text, and
do not expect stable machine-readable codes.

Related artifacts: `authentication/jampp-authentication.yml`,
`conventions/jampp-conventions.yml`, `rate-limits/jampp-rate-limits.yml`,
`vocabulary/jampp-reporting-vocabulary.yml`.
