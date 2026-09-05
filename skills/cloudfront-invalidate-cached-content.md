---
name: Invalidate cached CloudFront content
description: Purge objects from every CloudFront edge cache and poll the invalidation to completion — an irreversible, billable, rate-limited operation an agent should think twice about.
api: openapi/cloudfront-invalidations-api-openapi.yml
operations: [createInvalidation, getInvalidation, listInvalidations]
generated: '2026-09-05'
method: generated
source: openapi/cloudfront-invalidations-api-openapi.yml + conventions/cloudfront-conventions.yml + rate-limits/cloudfront-rate-limits.yml
---

# Invalidate cached CloudFront content

## Understand the consequence before calling

An invalidation **cannot be cancelled and cannot be undone.** Once submitted, the
matching objects are evicted from every edge location and the next viewer request
goes to the origin. AWS bills invalidation paths beyond the free monthly
allowance (1,000 paths). `/*` is a single path but purges everything and can
produce a large origin traffic spike.

This is an `acting` / consequence-bearing operation. Confirm intent before firing.

## Steps

1. **Submit the invalidation.** `createInvalidation`
   (`POST /2020-05-31/distribution/{DistributionId}/invalidation`) with an
   `InvalidationBatch` containing `Paths` and a **`CallerReference`**.
2. **Poll to completion.** `getInvalidation`
   (`GET /2020-05-31/distribution/{DistributionId}/invalidation/{Id}`) until
   `Status` is `Completed`. Typically a few minutes.
3. **Audit.** `listInvalidations` (`GET .../invalidation`) pages with
   `Marker` / `MaxItems` and shows recent invalidations for the distribution.

## Rules an agent must follow

- **Use a stable `CallerReference` for retries.** This is the one place
  CloudFront's idempotency contract genuinely protects you: replaying
  `createInvalidation` with the same `CallerReference` and an identical batch
  returns the ORIGINAL invalidation instead of creating and billing a second one.
  Generating a fresh reference on every retry defeats it. Coverage is `partial`
  across CloudFront (17 of 95 mutating operations) — see
  `conventions/cloudfront-conventions.yml`.
- **Same reference, different paths, is an error, not an update.** You get a 409
  `…AlreadyExists`-family error. Change the reference when the batch changes.
- **Respect the rate.** 150 paths or tags per second, and **1 wildcard
  invalidation per second**, per account. Exceeding it returns
  `TooManyInvalidationsInProgress` as HTTP 400 with no `Retry-After` — back off
  yourself.
- **Prefer versioned object names to invalidation.** Changing the object key
  (`/app.v37.js`) costs nothing and has no rate limit; invalidation costs money
  and is capped.
