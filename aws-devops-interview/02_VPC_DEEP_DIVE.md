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

### How NAT Gateway actually works

```text
private instance 10.0.2.10:49152
        |
        | route 0.0.0.0/0 -> nat-gateway
        v
NAT Gateway: source becomes its Elastic IP + translated source port
        |
        v
Internet Gateway -> destination

Reply matches translation state -> NAT -> 10.0.2.10:49152
```

The workload initiates the connection, which creates translation state. An internet client cannot initiate an arbitrary new connection “through” the NAT because no matching state or inbound listener exists. A NAT gateway is managed, zonal, IPv4 translation; it is not a firewall or inbound proxy. For IPv6, use native routing and an egress-only internet gateway when outbound-only initiation is required.

### NAT instance concept

A NAT instance is an EC2 instance configured to forward/translate traffic with source/destination checking disabled. It permits custom appliances/features but the customer owns patching, scaling, failover, bandwidth, route changes, and availability. NAT Gateway is normally preferred for managed scale and resilience; state the business requirement before choosing an instance.

### Elastic IP, ENI, and DHCP options

- **Elastic IP:** static public IPv4 address allocated to an account/Region and associated with a supported resource or network interface. Public IPv4 addresses are scarce and billable; allocate intentionally.
- **ENI:** virtual network interface containing private addresses, security groups, MAC address and attachment identity. Many managed services and pods consume ENIs/IPs.
- **DHCP option set:** supplies domain-name and DNS/NTP-related settings to VPC clients. It does not create DNS records or routing. Changing the associated set affects leases as clients renew.

## Route-table selection, precisely

1. The packet uses the route table associated with its source subnet; a subnet has exactly one effective table association.
2. Every VPC route table contains a `local` route for each VPC CIDR, enabling intra-VPC routing. It is more specific than a default internet route.
3. AWS chooses the longest matching destination prefix: `/32` beats `/24`, which beats `/0`.
4. If prefixes are identical, AWS route-priority rules determine static versus propagated choices; verify the current service-specific documentation rather than memorizing an incomplete hierarchy.
5. The target must be valid, every intermediate route domain must forward the packet, and the destination needs a return path.

Example:

```text
10.20.4.9 matches:
10.0.0.0/8   -> TGW
10.20.0.0/16 -> peering
10.20.4.0/24 -> firewall

Chosen: 10.20.4.0/24 -> firewall (longest prefix)
```

## Cross-account and multi-account networking

Resource sharing can place centrally owned subnets or Transit Gateway attachments in a multi-account design, but account ownership, route ownership, security-group references, DNS, logging, quotas, and incident responsibility remain explicit. A common pattern is separate workload accounts/VPCs connected to centrally governed TGW route domains, with network/security accounts operating inspection and hybrid attachments. Centralization reduces policy duplication but increases dependency and routing blast radius.

For multi-AZ design, create independent subnet tiers in each required AZ, keep zonal egress local where practical, distribute load-balancer targets/capacity, and test loss of an AZ. Multiple subnets alone do not create high availability if all workloads, NAT paths or dependencies remain in one AZ.

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

## Ordered production troubleshooting scenarios

In every case, capture the exact source, destination, port/protocol, timestamp, and whether the error is timeout, refusal, DNS failure, or application response.

### 1. EC2 cannot access the internet

1. Resolve the hostname and test the destination IP/port.
2. Determine whether the instance has public addressing or should use NAT.
3. Inspect its subnet route: public path to IGW or private path to a healthy NAT.
4. Check SG egress and NACL outbound plus ephemeral return rules.
5. Check Flow Logs, host route/proxy/firewall, then destination restrictions.

### 2. Private EC2 cannot download packages

1. Separate DNS failure from TCP/TLS/repository error using `dig` and `curl -v`.
2. Verify `0.0.0.0/0` goes to same-AZ NAT, or required service endpoints exist.
3. Inspect NAT error/connection metrics, Flow Logs, NACL and SG.
4. Verify repository allowlists, proxy, certificate time and OS package configuration.
5. Fix only the failed layer; do not assign a public IP as a shortcut.

### 3. ALB cannot reach EC2

1. Check target registration, AZ, target port and health reason.
2. Confirm application listens on the target address/port and health path returns expected code.
3. Allow target ingress from the ALB security group; allow ALB egress.
4. Check target-subnet NACL return ports and host firewall.
5. Inspect ALB access logs, Flow Logs and application logs at the same timestamp.

### 4. EC2 cannot reach RDS

1. Resolve the RDS endpoint from EC2 and confirm current port/engine.
2. Verify route/local connectivity or peering/TGW routes in both directions.
3. Allow RDS ingress from the EC2/application SG; verify source SG is actually attached.
4. Check NACLs, DB status, connection limit, TLS/authentication and database logs.
5. Distinguish network timeout from database authentication or pool exhaustion.

### 5. DNS is not resolving

1. Record query name/type, resolver address and exact response (`NXDOMAIN`, timeout, `SERVFAIL`).
2. Check VPC DNS support/hostnames and DHCP/Resolver configuration.
3. Inspect private hosted-zone association, overlapping zones and record correctness.
4. Check Resolver forwarding rules/endpoints, TCP/UDP 53 controls and loop risk.
5. Compare authoritative answer with client/OS/application caches and TTL.

### 6. Peered VPCs cannot communicate

1. Confirm peering is active and CIDRs do not overlap.
2. Add/verify forward and return routes in the correct subnet tables.
3. Check SG source CIDR or supported SG reference and both NACL directions.
4. Confirm DNS-resolution options if private names are used.
5. Prove the design is direct: peering cannot route transitively through a third VPC.

### 7. VPN path failure

1. Check both tunnel states, BGP sessions or static routes and recent device changes.
2. Verify AWS and on-premises route preference/advertisement in both directions.
3. Inspect firewall/NACL/SG and tunnel telemetry for drops/rekeys.
4. Test MTU/fragmentation and asymmetric routing with controlled probes.
5. Shift to a healthy redundant tunnel/path, then repair and test failback.

### 8. Asymmetric routing

1. Draw every forward and reverse hop, including TGW, firewall, NAT and VPN/DX.
2. Compare route tables and propagated/static routes on both sides.
3. Correlate flow/firewall logs by five-tuple; look for state seen on only one appliance.
4. Correct route symmetry or use supported appliance-mode architecture.
5. Test established and new sessions during controlled failover.

### 9. Security-group issue

1. Identify every ENI and SG on source, load balancer and destination.
2. Verify exact protocol/port and source/destination reference; SGs contain no deny rules.
3. Confirm the application uses the expected source identity/address after LB/NAT.
4. Use Reachability Analyzer/Flow Logs, then correct the narrow rule.
5. Add a connectivity contract test to prevent drift.

### 10. NACL issue

1. Locate NACLs on every traversed subnet.
2. Evaluate numbered rules from lowest upward for both directions.
3. Include ephemeral ports because NACLs are stateless.
4. Match Flow Log rejects and temporarily test only through reviewed narrow rules.
5. Simplify overly complex NACLs; retain SGs as primary workload control.

### 11. Route-table issue

1. Confirm which route table is associated with the source subnet.
2. Apply longest-prefix match to the real destination.
3. Check target/attachment state and intermediate TGW/firewall tables.
4. Verify the return route and rule out overlapping CIDRs.
5. Compare IaC/last-known-good configuration before a reviewed correction.

### 12. Endpoint issue

1. Confirm application DNS resolves to the intended interface endpoint or gateway route.
2. For interface endpoints, check subnet/AZ ENIs, private DNS and endpoint SG.
3. For gateway endpoints, check association with the source route table.
4. Evaluate identity, endpoint and resource policies plus KMS authorization.
5. Use CloudTrail/request IDs to distinguish network success from authorization failure.

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
