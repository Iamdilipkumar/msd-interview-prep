# 11 — CloudFront and WAF

## Edge request flow

```text
viewer -> WAF -> CloudFront cache behavior -> edge cache -> origin
                    | path/method/headers/cookies/query |
```

The cache key determines object variants. The origin request policy determines what CloudFront forwards. Forwarding everything reduces cache efficiency and can leak unnecessary viewer context to the origin.

## Design choices

| Concern | Approach |
|---|---|
| Private S3 origin | Origin Access Control plus restrictive bucket policy |
| Dynamic APIs | Separate behavior; forward only required data; tune timeouts |
| TLS | Viewer certificate/hostnames; origin TLS and host header correctness |
| Signed access | Signed URLs/cookies with trusted key groups |
| Updates | Versioned object names preferred; invalidate only when necessary |

## Cache miss or stale content

Inspect behavior match order, cache key, response cache headers, minimum/default/maximum TTL, error caching, invalidations, origin versions, and edge response headers. “Disable cache” is usually an expensive diagnostic, not a durable design.

## WAF

Start managed rules in count mode, inspect sampled requests/logs, then enforce with scoped exclusions. Add rate-based rules and narrowly designed custom rules. WAF is Layer 7 filtering; Shield addresses DDoS protections; neither replaces secure application code, authorization, or origin isolation.

## 403 workflow

Identify whether the response came from WAF, CloudFront, or origin using logs/request IDs. Check WAF rule match, alternate domain/certificate, signed URL, allowed methods, origin access policy, KMS/object permissions, and custom error caching.

## Memory hook

**Cache key chooses the variant; origin policy chooses forwarded context; WAF chooses permitted requests.**
