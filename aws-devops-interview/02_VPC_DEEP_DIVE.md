# 02 — VPC Deep Dive

## Packet-first mental model

```text
DNS -> source ENI -> subnet route table -> gateway/endpoint/NAT/TGW -> destination
         |                  |                                  |
       SG state          NACL stateless                     return path
```

When connectivity fails, prove: name resolution, source identity/IP, route in both directions, stateful security group, stateless NACL, listener, application, and response path.

## Core building blocks

| Component | Key point | Frequent trap |
|---|---|---|
| CIDR | Non-overlapping address plan enables growth and routing | Choosing a tiny or overlapping range |
| Subnet | AZ-scoped slice of VPC address space | Calling a subnet public merely because it has public IPs |
| Route table | Longest-prefix match; local route connects VPC subnets | Missing return route |
| Internet gateway | VPC attachment for public IPv4/IPv6 routing | Instance also needs public address and permitted traffic |
| NAT gateway | Outbound IPv4 translation for private subnets | It does not accept unsolicited inbound connections |
| Egress-only IGW | Outbound-only IPv6 path | Treating IPv6 like NATed IPv4 |
| Security group | Stateful allow rules on ENIs | Trying to write deny rules |
| NACL | Stateless ordered subnet rules | Forgetting ephemeral return ports |
| ENI | Network identity with addresses and security groups | Ignoring ENI/IP exhaustion |

### Public versus private

A public subnet has a route to an internet gateway. A resource is internet-reachable only if it also has a public address, permissive controls, and a listening service. Private workloads normally route required outbound IPv4 traffic through a NAT gateway or use private service endpoints.

## Security groups versus NACLs

| Dimension | Security group | Network ACL |
|---|---|---|
| Applied to | ENI/resource | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow and deny, numbered order |
| Return traffic | Automatically tracked | Explicitly permitted |
| Best use | Workload-level least privilege | Coarse subnet guardrail/emergency deny |

## VPC endpoints

- **Gateway endpoints:** route-table targets for S3 and DynamoDB; no NAT path needed.
- **Interface endpoints:** private ENIs powered by PrivateLink; evaluate endpoint DNS, security groups, subnet/AZ placement, policies, and hourly/data cost.
- Endpoint policy, identity policy, and service resource policy can all participate. A private path does not imply authorization.

## Connecting networks

| Option | Best fit | Main limitation |
|---|---|---|
| VPC peering | Simple one-to-one connectivity | Non-transitive; mesh grows rapidly; no overlapping CIDRs |
| Transit Gateway | Hub-and-spoke at scale | Route-domain and attachment cost/complexity |
| Site-to-Site VPN | Fast encrypted hybrid connectivity | Internet variability; use redundant tunnels |
| Direct Connect | Predictable private connectivity | Lead time; add resilient locations and VPN backup |
| PrivateLink | Expose a service, not an entire network | Producer/consumer design and per-endpoint cost |

```text
spokes -- TGW route domains -- shared services
  |             |
 prod       nonprod isolated
  |
DX + VPN -> on-premises
```

## DNS inside a VPC

Check VPC DNS attributes, DHCP options/Resolver, private hosted-zone association, split-horizon intent, Resolver inbound/outbound endpoints, rules, record TTL, and application caching. DNS success does not prove transport connectivity.

## NAT and cost/resilience

Deploy NAT gateways per AZ when zonal independence is required and route each private subnet to its same-AZ NAT. Cross-AZ routing can add dependency and data charges. Reduce NAT traffic with endpoints where justified. Track port exhaustion, connection count, errors, bytes, and flow logs.

## Troubleshooting workflow

```text
1 DNS result          dig/nslookup
2 source/destination  IP, ENI, port, protocol
3 forward route       route table, TGW propagation/static route
4 policy              SG both sides, NACL both directions, endpoint policy
5 path evidence       Reachability Analyzer, VPC Flow Logs
6 destination         target health, listener, process, host firewall
7 return path         asymmetric route, NAT/firewall state
8 packet proof        tcpdump where controllable
```

Flow Logs show accepted/rejected network-flow metadata, not application payload. Reachability Analyzer models configuration; it does not generate packets or prove the application is healthy.

## Senior design questions

### How would you address CIDR exhaustion?

Measure free addresses by subnet and ENI consumers; reclaim unused interfaces; add secondary VPC CIDRs and subnets; tune pod networking where relevant; then migrate workloads deliberately. Never “resize” a subnet in place. Build IP allocation into capacity planning.

### How would you isolate environments?

Prefer account boundaries, separate VPCs, centralized inspection only where required, explicit TGW route tables, no broad default routes between trust zones, private endpoints, and logged DNS/network controls. Validate that shared services do not create transitive bypasses.

### How do you diagnose intermittent connection timeout?

Correlate timestamps with NAT port metrics, NACL ephemeral ports, load-balancer target health, DNS answers/TTL, connection tracking, cross-zone paths, and downstream saturation. Compare a failed five-tuple against Flow Logs and application traces.

## Memory hooks

- **RSPAR:** Resolve, Source, Path, Allow, Return.
- **SG remembers; NACL forgets.**
- **Route chooses path; policy permits path; application completes path.**

## Official references

- [Amazon VPC routing](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)
- [Security groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
- [VPC endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html)
