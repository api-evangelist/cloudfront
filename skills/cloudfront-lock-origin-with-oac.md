---
name: Lock a CloudFront origin down with origin access control
description: Create an origin access control and attach it to a distribution's origin so the origin only accepts signed requests from CloudFront — the modern replacement for origin access identity.
api: openapi/cloudfront-originaccesscontrol-api-openapi.yml
operations: [createOriginAccessControl, listOriginAccessControls, getOriginAccessControl, getDistributionConfig, updateDistribution, deleteOriginAccessControl]
generated: '2026-09-05'
method: generated
source: openapi/cloudfront-originaccesscontrol-api-openapi.yml + openapi/cloudfront-distributions-api-openapi.yml + smithy/cloudfront-2020-05-31.json
---

# Lock a CloudFront origin down with origin access control

Origin access control (OAC) makes CloudFront sign its requests to the origin with
SigV4, so the origin can refuse anything that did not come through CloudFront.
It supersedes origin access identity (OAI) — use OAC for anything new. Both
surfaces are still callable and **neither is marked deprecated in the contract**
(see `lifecycle/cloudfront-lifecycle.yml`), so the contract will not warn you off
the legacy path.

## Steps

1. **Check for an existing control first.** `listOriginAccessControls`
   (`GET /2020-05-31/origin-access-control`). The account quota is 100
   (`TooManyOriginAccessControls`), so reuse before creating.
2. **Create the control.** `createOriginAccessControl`
   (`POST /2020-05-31/origin-access-control`) with an
   `OriginAccessControlConfig`: `Name`, `OriginAccessControlOriginType`,
   `SigningBehavior`, `SigningProtocol`. Keep the returned `Id` and `ETag`.
3. **Read the distribution config.** `getDistributionConfig`
   (`GET /2020-05-31/distribution/{Id}/config`) — keep its `ETag`.
4. **Attach it.** Set `OriginAccessControlId` on the target entry in
   `Origins.Items[]`, then `updateDistribution`
   (`PUT /2020-05-31/distribution/{Id}/config`) with `If-Match: <ETag>`.
5. **Grant the origin side.** The origin's own policy (for S3, the bucket policy)
   must allow the CloudFront service principal for that distribution ARN. This is
   **not** a CloudFront API operation — CloudFront cannot write your origin's
   policy for you, and skipping it produces 403s at the edge.
6. **Verify.** `getOriginAccessControl` (`GET /2020-05-31/origin-access-control/{Id}`)
   and re-read the distribution to confirm the attachment.

## Rules an agent must follow

- **Order matters.** Attach the OAC and update the origin policy before removing
  any public access from the origin, or you will take the site down between steps.
- **`deleteOriginAccessControl` fails while in use.** You get a 409
  `OriginAccessControlInUse`. Detach it from every distribution first.
- **This is reversible, but only by hand.** There is no "undo attach": you
  re-apply the previous `DistributionConfig`, which means you must have kept the
  body from step 3. See `conventions/cloudfront-conventions.yml` → `reversibility`.
