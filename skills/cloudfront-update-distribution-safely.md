---
name: Update a CloudFront distribution safely
description: Read the current distribution config, mutate it, and write it back under an ETag precondition so a concurrent change cannot be lost — then wait for the change to deploy.
api: openapi/cloudfront-distributions-api-openapi.yml
operations: [getDistributionConfig, updateDistribution, getDistribution, listDistributions]
generated: '2026-09-05'
method: generated
source: openapi/cloudfront-distributions-api-openapi.yml + conventions/cloudfront-conventions.yml + smithy/cloudfront-2020-05-31.json
---

# Update a CloudFront distribution safely

CloudFront has **no partial update**. `updateDistribution` replaces the whole
`DistributionConfig`. Sending a config you did not first read will silently drop
every field you omitted. Always read, mutate, write.

## Before you start

- Sign every request with **AWS SigV4**, service name `cloudfront`, region
  `us-east-1` — CloudFront is global and always signs `us-east-1`.
- Requests and responses are **XML**, not JSON. See
  `conventions/cloudfront-conventions.yml`.

## Steps

1. **Find the distribution.** `listDistributions` (`GET /2020-05-31/distribution`).
   Page with `Marker` / `MaxItems`; loop while `IsTruncated` is true, passing the
   previous `NextMarker`.
2. **Read the current config.** `getDistributionConfig`
   (`GET /2020-05-31/distribution/{Id}/config`). **Keep the `ETag` from the
   response header — you cannot write without it, and you cannot roll back
   without the config body either.** CloudFront exposes no configuration history
   operation, so this response is your only snapshot.
3. **Mutate the config in place.** Change only the fields you intend to change.
   Leave `CallerReference` exactly as returned — altering it does not create a new
   distribution on an update, it just breaks the replay contract.
4. **Write it back.** `updateDistribution`
   (`PUT /2020-05-31/distribution/{Id}/config`) with `If-Match: <ETag>` and the
   full config body.
5. **Wait for deployment.** `getDistribution` (`GET /2020-05-31/distribution/{Id}`)
   until `Status` is `Deployed`. No CloudFront operation blocks; you must poll.
   Propagation is typically minutes, not seconds.

## Rules an agent must follow

- **`If-Match` is mandatory.** A missing or stale ETag returns
  `PreconditionFailed` (HTTP 412). Do not retry with the same ETag — go back to
  step 2 and re-read.
- **Idempotency is partial here.** `updateDistribution` carries `CallerReference`,
  so an identical replay is safe; but the ETag will have moved after the first
  success, so the replay fails on the precondition instead. Treat 412 after a
  timeout as "probably already applied" and re-read to confirm — never assume
  failure.
- **Errors are XML.** Parse `ErrorResponse/Error/Code`; there is no
  `application/problem+json`. Catalog: `errors/cloudfront-error-codes.yml`.
- **Quota errors are HTTP 400, not 429.** `TooManyDistributionCNAMEs`,
  `TooManyCacheBehaviors`, `TooManyOrigins` and 54 siblings mean you exceeded a
  published quota; there is no `Retry-After` and retrying will not help.
  See `rate-limits/cloudfront-rate-limits.yml`.
- **Rolling back means re-applying the config from step 2.** If you did not keep
  it, you cannot roll back. For risky changes prefer continuous deployment
  (`UpdateDistributionWithStagingConfig`), which is CloudFront's only first-class
  rollback path.
