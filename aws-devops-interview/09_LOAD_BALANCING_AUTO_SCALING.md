# 09 — Load Balancing and Auto Scaling

## Choose by protocol and need

| Service | Fit | Strength |
|---|---|---|
| ALB | HTTP/HTTPS/gRPC | Layer-7 routing, redirects, authentication integration |
| NLB | TCP/UDP/TLS | High-performance Layer 4, static IP needs, source IP behavior |
| GWLB | Virtual network appliances | Transparent insertion using GENEVE |

## Request flow

```text
client -> DNS -> listener -> rule -> target group -> target port -> app
                    TLS       health check        SG + readiness
```

Troubleshoot each hop. A healthy load balancer with no healthy targets cannot serve. A healthy target can still return application errors on routes not covered by the health check.

## Common status-code diagnosis

| Symptom | Investigate first |
|---|---|
| 502 | Target closed/malformed response, TLS mismatch, app crash |
| 503 | No healthy/available targets, capacity, target registration |
| 504 | Target response timeout, dependency/query latency, timeout chain |
| Redirect loop | Proxy headers, TLS termination, application scheme awareness |

## Health checks

Use a lightweight endpoint that proves the instance can safely receive traffic. Readiness may test critical dependencies, but an overly deep check can remove all targets during a shared dependency issue. Separate liveness, readiness, and external SLO monitoring.

## Auto Scaling

| Policy | Use |
|---|---|
| Target tracking | Maintain a demand-per-capacity target |
| Step scaling | Different responses to alarm severity |
| Scheduled | Predictable demand |
| Predictive | Recurring patterns with usable history |

Scale on the bottleneck. CPU is useful only if demand correlates with CPU. ALB requests per target, queue backlog per worker, or custom concurrency can be better. Set min/max, warm-up, cooldown, health replacement, mixed instances, capacity rebalance, and downstream protection.

## Resilient rollout

Use launch-template versions, small canary/instance refresh waves, minimum healthy capacity, warm-up, lifecycle hooks, alarms, and automatic rollback. Verify that scaling can actually obtain capacity and subnet IPs.

## Memory hook

**LB distributes; health admits; ASG supplies.** They solve different problems.
