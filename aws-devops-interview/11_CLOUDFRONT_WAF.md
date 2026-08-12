# 11 — CloudFront and WAF

## Edge request flow

```text
viewer -> WAF -> CloudFront cache behavior -> edge cache -> origin
                    | path/method/headers/cookies/query |
```

The cache key determines object variants. The origin request policy determines what CloudFront forwards. Forwarding everything reduces cache efficiency and can leak unnecessary viewer context to the origin.

An origin can be S3, ALB, API endpoint or custom HTTP service. Ordered cache behaviors match paths and select origin, allowed methods, protocol, cache/origin policies and edge functions. Cache TTL is influenced by behavior policy and origin cache headers. Invalidations remove cached paths but cost/time and race behavior make versioned object keys a better normal release mechanism.

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

Signed URLs suit individual objects; signed cookies suit access to groups of paths. Both require trusted signing keys and expiry policy. Viewer headers, cookies and query strings affect cache fragmentation; forward only what the origin needs. Use modern TLS policies/certificates and decide whether origin TLS validates the same or a separate hostname.

For **502**, inspect origin DNS/TLS/protocol/connection failures. For **504**, inspect origin response timeout and downstream latency. Correlate CloudFront request ID, standard/real-time logs as justified, WAF logs and origin logs before changing cache behavior.

## 403 workflow

Identify whether the response came from WAF, CloudFront, or origin using logs/request IDs. Check WAF rule match, alternate domain/certificate, signed URL, allowed methods, origin access policy, KMS/object permissions, and custom error caching.

## Memory hook

**Cache key chooses the variant; origin policy chooses forwarded context; WAF chooses permitted requests.**
