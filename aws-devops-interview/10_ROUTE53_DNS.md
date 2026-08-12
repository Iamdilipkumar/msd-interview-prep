# 10 — Route 53 and DNS

## Resolution path

```text
application cache -> OS cache -> recursive resolver -> root/TLD -> authoritative zone
```

TTL controls cache lifetime, not instantaneous cutover. Existing connections can outlive DNS records. Negative answers may also be cached.

A public hosted zone is authoritative on the internet after correct registrar delegation. A private hosted zone answers only through associated VPC/Resolver contexts. `A` maps a name to IPv4, `AAAA` to IPv6, and `CNAME` aliases one non-apex name to another. A Route 53 Alias is an AWS extension that can target supported AWS resources and work at the zone apex.

## Routing policies

| Policy | Use | Trap |
|---|---|---|
| Simple | One uncomplicated response | No health-based traffic management |
| Weighted | Canary or controlled split | Resolver caching means approximate distribution |
| Latency | Direct users to lower-latency Region | Not a guarantee of application health |
| Failover | Active/passive with health evaluation | Dependency and false-positive design |
| Geolocation | Location-based policy/compliance | Needs default behavior |
| Multivalue | Multiple healthy answers | Not a load-balancer replacement |

Alias records integrate with supported AWS resources and can be used at the zone apex. CNAME records cannot be used at the apex under DNS rules.

## Hybrid DNS

Route 53 Resolver inbound endpoints let on-premises resolvers query selected AWS namespaces; outbound endpoints and forwarding rules send selected queries toward on-premises DNS. Deploy endpoints redundantly, associate rules correctly, allow TCP/UDP 53, and prevent forwarding loops.

Route 53 health checks can evaluate endpoints or calculated checks and influence supported routing records. They do not prove every dependency or data plane. Design thresholds, regions, false-positive behavior and a recovery control plane independent enough to operate during the failure.

## Failure diagnosis

1. Capture queried name/type, resolver, answer, TTL, and authoritative response.
2. Check delegation/name servers and DNSSEC where used.
3. Check public/private hosted-zone overlap and VPC association.
4. Verify record/routing policy/health-check state.
5. Test the resolved target independently.
6. Inspect application/OS caching and connection reuse.

## Senior principle

DNS failover is one component of DR. Data readiness, service capacity, certificates, secrets, dependency endpoints, client retry/caching, and failback must also work.
