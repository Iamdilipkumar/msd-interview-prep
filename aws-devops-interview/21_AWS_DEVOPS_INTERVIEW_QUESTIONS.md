# 21 — AWS DevOps Interview Questions

Exactly 150 high-value questions. Answer aloud before reading. “Strong answer” is a compact outline to expand with assumptions and tradeoffs; it is not a script to memorize.

## VPC and networking — 30

### Q001 — What makes a subnet public?
- **Testing:** Routing fundamentals. **30-second answer:** Its route table has a route to an internet gateway; resources also need public addressing and permitted traffic.
- **Strong answer:** Separate subnet routing from resource reachability: IGW attachment/route, public IPv4 or IPv6 path, SG/NACL, listener, and return path all matter.
- **Follow-ups:** Can private IP reach internet? Does SG allow deny? **Angle:** Trace one packet. **Hook:** Public is a route, not a label.

### Q002 — Security group versus NACL?
- **Testing:** Stateful versus stateless control. **30-second answer:** SG is stateful allow-only at ENI; NACL is ordered stateless allow/deny at subnet.
- **Strong answer:** Use SGs for workload least privilege and NACLs as coarse guardrails; NACLs need explicit return/ephemeral-port rules.
- **Follow-ups:** Rule evaluation? Return traffic? **Angle:** Check both directions. **Hook:** SG remembers; NACL forgets.

### Q003 — How does route selection work?
- **Testing:** Packet routing. **30-second answer:** Most-specific matching destination prefix wins; static/propagated precedence then matters for equal prefixes.
- **Strong answer:** Identify source subnet table, destination, longest prefix, target health, intermediate route domains, and symmetric return path.
- **Follow-ups:** Local route? Blackhole? **Angle:** Forward and return tables. **Hook:** Most specific first.

### Q004 — NAT gateway versus internet gateway?
- **Testing:** Egress design. **30-second answer:** IGW enables internet routing for publicly addressed resources; NAT translates private IPv4 outbound flows and rejects unsolicited inbound initiation.
- **Strong answer:** Place zonal NAT per AZ for resilience, route private subnets locally, monitor ports/errors, and use endpoints to reduce NAT dependency/cost.
- **Follow-ups:** IPv6? HA? **Angle:** Trace address translation. **Hook:** IGW routes; NAT translates.

### Q005 — Why deploy one NAT gateway per AZ?
- **Testing:** Zonal resilience/cost. **30-second answer:** It removes cross-AZ egress dependency and can reduce cross-AZ data transfer.
- **Strong answer:** Each private subnet routes to its AZ’s NAT; weigh availability and traffic cost against extra hourly cost.
- **Follow-ups:** Failure behavior? Alternatives? **Angle:** Inspect subnet route ownership. **Hook:** Keep zonal traffic zonal.

### Q006 — Gateway versus interface endpoint?
- **Testing:** Private service access. **30-second answer:** Gateway endpoints are route targets for S3/DynamoDB; interface endpoints are PrivateLink ENIs for supported services.
- **Strong answer:** Compare DNS, SG, endpoint policy, subnet/AZ placement, service support, hourly and data cost.
- **Follow-ups:** Does endpoint grant access? **Angle:** Network and authorization separately. **Hook:** Route target versus ENI.

### Q007 — Peering versus Transit Gateway?
- **Testing:** Network topology. **30-second answer:** Peering is simple non-transitive point-to-point; TGW is scalable hub-and-spoke with route domains.
- **Strong answer:** Select on connection count, segmentation, inspection, hybrid routing, throughput/cost, and operational ownership; neither supports overlapping CIDRs directly.
- **Follow-ups:** Transitivity? Appliance mode? **Angle:** Draw route domains. **Hook:** Pair versus hub.

### Q008 — PrivateLink versus peering?
- **Testing:** Service versus network exposure. **30-second answer:** PrivateLink exposes a service privately; peering connects VPC address spaces bidirectionally through routes.
- **Strong answer:** PrivateLink reduces lateral reach and CIDR coupling but adds endpoint/service cost and producer design; use it for consumer-to-service access.
- **Follow-ups:** DNS? Initiation direction? **Angle:** Define what must be reachable. **Hook:** Service, not subnet.

### Q009 — Direct Connect versus VPN?
- **Testing:** Hybrid tradeoffs. **30-second answer:** DX offers private predictable connectivity; VPN is encrypted over internet and faster to establish.
- **Strong answer:** Use redundant DX locations/connections and often VPN backup; address BGP, encryption needs, bandwidth, lead time, and failure testing.
- **Follow-ups:** Is DX encrypted? **Angle:** Test each redundant path. **Hook:** Predictability versus speed.

### Q010 — How do you troubleshoot a timeout between EC2 instances?
- **Testing:** Layered diagnosis. **30-second answer:** Resolve DNS, verify source/destination/port, routes both ways, SG, NACL, listener, host firewall, then app.
- **Strong answer:** Use Reachability Analyzer for modeled path, Flow Logs for five-tuple evidence, and packet capture where controlled; correlate exact timestamps.
- **Follow-ups:** Refused versus timeout? **Angle:** RSPAR workflow. **Hook:** Resolve, source, path, allow, return.

### Q011 — What do VPC Flow Logs prove?
- **Testing:** Evidence limits. **30-second answer:** They record flow metadata and accept/reject status at selected interfaces; not payload or application health.
- **Strong answer:** Match five-tuple/time/action, account for aggregation and logging configuration, then combine with route/config and application logs.
- **Follow-ups:** Can they show DNS content? **Angle:** Know signal boundaries. **Hook:** Flow, not packet body.

### Q012 — What does Reachability Analyzer do?
- **Testing:** Tool accuracy. **30-second answer:** It statically analyzes AWS network configuration for a modeled path; it does not send traffic.
- **Strong answer:** Use it to identify route/security/config blockers, then verify DNS, runtime listener, host policy, and application separately.
- **Follow-ups:** Intermittent failures? **Angle:** Configuration proof, runtime proof. **Hook:** Model, not probe.

### Q013 — How do ephemeral ports affect NACLs?
- **Testing:** Stateless return path. **30-second answer:** Return traffic commonly uses client ephemeral ports, which stateless NACLs must explicitly permit.
- **Strong answer:** Identify who initiates, protocol/port, OS/load-balancer port behavior, and define both ingress and egress rules without overly broad exposure.
- **Follow-ups:** SG behavior? **Angle:** Reverse the five-tuple. **Hook:** NACL needs the reply rule.

### Q014 — What is asymmetric routing risk?
- **Testing:** Stateful path awareness. **30-second answer:** Request and response traverse different stateful devices, so return traffic may lack connection state and be dropped.
- **Strong answer:** Inspect TGW/firewall/NAT route symmetry, appliance mode, and propagation; correlate both directions in logs.
- **Follow-ups:** How fix? **Angle:** Draw forward and return hops. **Hook:** State must see both ways.

### Q015 — How do you plan VPC CIDRs?
- **Testing:** Scale planning. **30-second answer:** Allocate non-overlapping ranges with growth for subnets, AZs, pods, endpoints, and hybrid networks using IPAM.
- **Strong answer:** Forecast ENI-heavy services, reserve expansion, segment route domains, document ownership, and prevent overlaps through organization controls.
- **Follow-ups:** Exhaustion recovery? **Angle:** Measure actual IP consumers. **Hook:** Addresses are capacity.

### Q016 — How do you handle subnet IP exhaustion?
- **Testing:** Production recovery. **30-second answer:** Find ENI consumers, reclaim leaks, add secondary VPC CIDR/new subnets, and migrate capacity safely.
- **Strong answer:** Alarm on available IPs; for EKS evaluate pod networking/prefix delegation; never assume a subnet can be resized in place.
- **Follow-ups:** Which services consume ENIs? **Angle:** Inventory before change. **Hook:** Add subnet, not resize.

### Q017 — How does DNS work in a VPC?
- **Testing:** Resolver architecture. **30-second answer:** Workloads normally query Route 53 Resolver; VPC attributes, private zones, Resolver rules/endpoints, and caches shape answers.
- **Strong answer:** Separate resolution from connectivity; inspect zone association, split-horizon overlap, DHCP settings, forwarding loops, TTL and app caching.
- **Follow-ups:** Hybrid names? **Angle:** Query authoritative and recursive layers. **Hook:** Answer first, path second.

### Q018 — Private hosted zone problem diagnosis?
- **Testing:** DNS troubleshooting. **30-second answer:** Verify VPC association, exact zone/record, Resolver path, overlapping zones, record type, and client cache.
- **Strong answer:** Query from affected VPC and a known-good resolver, inspect forwarding rules and zone precedence, then test target independently.
- **Follow-ups:** NXDOMAIN caching? **Angle:** Capture resolver and TTL. **Hook:** Zone association before route.

### Q019 — How would you centralize egress?
- **Testing:** Inspection architecture. **30-second answer:** Route spoke egress through controlled TGW route domains to inspection/NAT, while preserving symmetry and zonal resilience.
- **Strong answer:** Compare decentralized NAT/endpoints versus centralized inspection on cost, latency, blast radius, scaling, route complexity, and ownership.
- **Follow-ups:** Cross-AZ cost? **Angle:** Failure-domain diagram. **Hook:** Central control, central dependency.

### Q020 — How do you segment a multi-account network?
- **Testing:** Blast radius. **30-second answer:** Separate VPCs/accounts, attach through TGW with distinct route tables, and expose shared services explicitly.
- **Strong answer:** No broad transitive defaults; isolate prod/nonprod/security zones, control propagation, centralize DNS selectively, and validate paths continuously.
- **Follow-ups:** Shared services risk? **Angle:** Route domains are trust domains. **Hook:** Attach does not mean route.

### Q021 — IPv4 versus IPv6 egress?
- **Testing:** Dual-stack understanding. **30-second answer:** IPv4 private egress often uses NAT; IPv6 uses globally unique addresses with an egress-only IGW for outbound-only initiation.
- **Strong answer:** Update routes, SG/NACL, DNS, logging, application binding, and third-party allowlists for both protocols; dual stack doubles paths to test.
- **Follow-ups:** NAT64? **Angle:** Test A and AAAA independently. **Hook:** IPv6 is not NATed IPv4.

### Q022 — How do you restrict S3 access to a VPC endpoint?
- **Testing:** Layered policy. **30-second answer:** Use bucket policy conditions plus endpoint and identity policies, with careful exceptions for trusted administration/services.
- **Strong answer:** Test exact principal/action/resource/source endpoint and account for KMS; a restrictive bucket condition can break AWS service workflows.
- **Follow-ups:** Console access? **Angle:** Simulate every access path. **Hook:** Path condition plus identity.

### Q023 — Why can ping fail while HTTPS works?
- **Testing:** Protocol distinction. **30-second answer:** ICMP may be blocked or unsupported while TCP 443 is routed, allowed, and served.
- **Strong answer:** Test the actual application protocol; security controls and load balancers handle protocols differently.
- **Follow-ups:** Better tools? **Angle:** `curl`, TCP connect, traces. **Hook:** Test what users use.

### Q024 — What is a bastion alternative?
- **Testing:** Secure operations. **30-second answer:** Systems Manager Session Manager provides audited access without public IPs or inbound SSH when prerequisites are met.
- **Strong answer:** Use federated identity, narrow session permissions, logging/encryption, private endpoints/egress, patched agent, and break-glass procedure.
- **Follow-ups:** If SSM fails? **Angle:** Verify agent/IAM/endpoints. **Hook:** Identity-mediated access.

### Q025 — How would you debug intermittent DNS latency?
- **Testing:** Evidence correlation. **30-second answer:** Measure client, cache, Resolver, CoreDNS if present, forwarding endpoints, upstream servers, and network loss by timestamp.
- **Strong answer:** Compare cached/uncached queries, UDP/TCP fallback, saturation, forwarding loops, TTL behavior, and affected AZs.
- **Follow-ups:** Large response? **Angle:** Preserve query name/type/server. **Hook:** Cache, resolver, authority.

### Q026 — What is split-horizon DNS?
- **Testing:** Naming design. **30-second answer:** The same name returns different answers based on resolver/network context, often public versus private zones.
- **Strong answer:** Use deliberately; document zone association and hybrid forwarding, avoid surprising overlap, and test from each resolution domain.
- **Follow-ups:** Failure modes? **Angle:** Compare answers from contexts. **Hook:** Same name, different view.

### Q027 — How do you make hybrid DNS resilient?
- **Testing:** Enterprise integration. **30-second answer:** Deploy Resolver endpoints across AZs, redundant on-prem resolvers, scoped forwarding rules, monitored capacity, and loop-free fallback.
- **Strong answer:** Define namespace authority, TCP/UDP 53 paths, security controls, query logging, failure behavior, and test loss of each endpoint.
- **Follow-ups:** Conditional forwarding? **Angle:** Draw authority chain. **Hook:** Know who owns each suffix.

### Q028 — What causes NAT data cost surprises?
- **Testing:** Cost reasoning. **30-second answer:** High internet/service traffic through NAT, cross-AZ paths, and duplicated transfer can dominate.
- **Strong answer:** Attribute bytes by flow/service, add endpoints where economics/security justify, keep same-AZ routing, cache or redesign chatty calls.
- **Follow-ups:** Endpoint costs? **Angle:** Compare total cost, not hourly only. **Hook:** Follow the bytes.

### Q029 — Design ingress for private workloads.
- **Testing:** Layered architecture. **30-second answer:** Public ALB/NLB spans public subnets while targets run privately; SG references constrain LB-to-target traffic.
- **Strong answer:** Add WAF/CloudFront as required, TLS/certificates, health checks, zonal capacity, access logs, egress/dependency paths, and DDoS posture.
- **Follow-ups:** NLB SG/source IP? **Angle:** Draw listener-to-target. **Hook:** Public front, private target.

### Q030 — Your network change caused outage; what now?
- **Testing:** Incident discipline. **30-second answer:** Freeze changes, define impact, roll back the known change if safe, preserve flow/config evidence, and verify restoration.
- **Strong answer:** Compare before/after routes/policies, check both directions, communicate, then root-cause and add IaC plan/reachability tests and staged rollout.
- **Follow-ups:** When not rollback? **Angle:** Smallest reversible mitigation. **Hook:** Stabilize, prove, prevent.

## IAM — 20

### Q031 — Explain IAM policy evaluation.
- **Testing:** Authorization fundamentals. **30-second answer:** Applicable allows grant unless an explicit deny in any relevant policy plane overrides; otherwise implicit deny.
- **Strong answer:** Evaluate principal, action, resource, context across identity/resource policy, boundary, session, SCP/RCP, endpoint and service-specific policies.
- **Follow-ups:** Cross-account? **Angle:** Trace exact request. **Hook:** Deny beats allow.

### Q032 — Role versus user?
- **Testing:** Identity design. **30-second answer:** A user is a long-lived identity; a role is assumed to obtain temporary session credentials.
- **Strong answer:** Prefer federation and roles for humans/workloads, narrow trust and permissions, short sessions, and avoid static keys.
- **Follow-ups:** Instance profile? **Angle:** Identify credential source. **Hook:** Roles are sessions.

### Q033 — Trust policy versus permissions policy?
- **Testing:** AssumeRole model. **30-second answer:** Trust says who may assume; permissions say what the resulting role session may do.
- **Strong answer:** Caller permission, role trust conditions, session policy, boundary and SCP can all affect the result.
- **Follow-ups:** Resource-based delegation? **Angle:** Separate entry from authority. **Hook:** Who enters; what they do.

### Q034 — What is a permissions boundary?
- **Testing:** Delegation guardrail. **30-second answer:** It caps permissions identity policies can grant; it grants no permissions itself.
- **Strong answer:** Use it for delegated role creation while recognizing resource policies/session semantics and explicit denies need full evaluation.
- **Follow-ups:** Boundary versus SCP? **Angle:** Intersection of effective permissions. **Hook:** Ceiling, not grant.

### Q035 — What is an SCP?
- **Testing:** Organization governance. **30-second answer:** It limits maximum permissions in member accounts; it does not grant access.
- **Strong answer:** Apply carefully by OU, test service-linked/emergency paths, and diagnose explicit organizational denies from the actual account/session.
- **Follow-ups:** Management account? **Angle:** Guardrail blast radius. **Hook:** Org ceiling.

### Q036 — How do you achieve least privilege?
- **Testing:** Practical governance. **30-second answer:** Start from required actions/resources/conditions, use short-lived roles, observe actual use, then continuously refine.
- **Strong answer:** Split runtime/deploy/admin roles; use Access Analyzer, CloudTrail, policy simulation, access last-used data, review and expiry.
- **Follow-ups:** New workload bootstrap? **Angle:** Test business operations. **Hook:** Need, scope, observe, shrink.

### Q037 — How do you troubleshoot AccessDenied?
- **Testing:** Systematic method. **30-second answer:** Capture session principal, action, resource, context/request ID, then evaluate every applicable policy plane and conditions.
- **Strong answer:** Use CloudTrail and simulator, check boundary/session/SCP/resource/endpoint/KMS policy, correct the smallest missing permission.
- **Follow-ups:** No CloudTrail event? **Angle:** Confirm endpoint/account/region. **Hook:** PARC—principal, action, resource, context.

### Q038 — Explain `iam:PassRole`.
- **Testing:** Privilege delegation. **30-second answer:** It lets a principal configure a service to use a role; it does not assume that role.
- **Strong answer:** Restrict role ARN/path and `iam:PassedToService`; otherwise users may pass a powerful role to exploitable compute.
- **Follow-ups:** CloudTrail event? **Angle:** Inspect create/update API. **Hook:** Pass to service.

### Q039 — How does cross-account access work?
- **Testing:** Dual-side policy. **30-second answer:** A trusted principal assumes a role or accesses a resource whose policy permits it, with both accounts’ controls satisfied.
- **Strong answer:** Prefer role sessions, narrow principal/org/external ID/source conditions, resource permissions, logging, and explicit ownership.
- **Follow-ups:** S3 resource policy? **Angle:** Draw caller and resource account. **Hook:** Both sides agree.

### Q040 — Why use external ID?
- **Testing:** Confused deputy. **30-second answer:** It helps a third-party role distinguish the intended customer relationship and resist confused-deputy attacks.
- **Strong answer:** The third party should generate/manage unique values; still constrain principal, permissions, sessions, and audit.
- **Follow-ups:** Is it secret? **Angle:** Test trust conditions. **Hook:** Customer binding, not password.

### Q041 — What is role chaining?
- **Testing:** STS session nuance. **30-second answer:** An assumed role uses its session to assume another role, with session-duration and policy implications.
- **Strong answer:** Minimize chains, understand source identity/session tags, max duration, trust, and auditability; prefer direct federation where possible.
- **Follow-ups:** Duration limit? **Angle:** Inspect CloudTrail session issuer. **Hook:** Session assumes session.

### Q042 — Workload identity on EC2?
- **Testing:** Credential hygiene. **30-second answer:** Attach an instance profile role and retrieve temporary credentials through IMDS; avoid static keys.
- **Strong answer:** Require IMDSv2, limit hop/access, scope role, monitor sessions, and separate applications when host sharing weakens isolation.
- **Follow-ups:** SSRF risk? **Angle:** Verify credential provider chain. **Hook:** Profile, not key file.

### Q043 — EKS Pod Identity versus IRSA?
- **Testing:** Current EKS identity. **30-second answer:** Both map service accounts to IAM roles; Pod Identity uses EKS associations/Auth agent, IRSA uses OIDC web identity.
- **Strong answer:** Choose based on SDK/add-on support, trust/cluster portability, operating model, and cross-account pattern; avoid broad node roles.
- **Follow-ups:** Blue/green impact? **Angle:** Inspect service account/session. **Hook:** Agent association versus OIDC token.

### Q044 — ECS task role versus execution role?
- **Testing:** Runtime boundary. **30-second answer:** Task role is for application AWS calls; execution role is for ECS agent actions such as image pull/logs/secrets.
- **Strong answer:** Scope each task/service separately, check stopped reason and CloudTrail using the correct principal.
- **Follow-ups:** EC2 container role? **Angle:** Who made the call? **Hook:** App task; agent execution.

### Q045 — How should CI access AWS?
- **Testing:** Supply-chain identity. **30-second answer:** Federate via OIDC to short-lived, environment-specific roles; do not store access keys.
- **Strong answer:** Constrain repository/branch/environment claims, session duration, permissions, protected approvals, and CloudTrail attribution.
- **Follow-ups:** Fork PR? **Angle:** Treat source context as untrusted. **Hook:** Exchange identity, not secret.

### Q046 — Resource policy versus identity policy?
- **Testing:** Policy placement. **30-second answer:** Identity policy attaches to a principal; resource policy attaches to a supported resource and names principals.
- **Strong answer:** Effective access depends on same/cross-account semantics plus boundaries, SCPs, conditions, and explicit denies.
- **Follow-ups:** Which services? **Angle:** Inspect both sides. **Hook:** Caller versus object.

### Q047 — What are session tags?
- **Testing:** ABAC/federation. **30-second answer:** Attributes passed into a role session can drive policy conditions and attribution.
- **Strong answer:** Control who may set/transit tags, source values from trusted identity claims, and avoid user-controlled privilege escalation.
- **Follow-ups:** Transitive tags? **Angle:** Validate trust and tag keys. **Hook:** Trusted attributes on session.

### Q048 — How do you design break-glass access?
- **Testing:** Resilience and governance. **30-second answer:** A tightly controlled, monitored emergency role with strong authentication, short duration, approval/alerting, and post-use review.
- **Strong answer:** Keep it independent of likely failure paths, test access, restrict actions, log immutably, rotate controls, and never use routinely.
- **Follow-ups:** If federation is down? **Angle:** Exercise the procedure. **Hook:** Available, narrow, noisy.

### Q049 — IAM authorization with KMS?
- **Testing:** Multi-policy interaction. **30-second answer:** IAM permission may not suffice; KMS key policy/grants and encryption context also govern use.
- **Strong answer:** Check correct key/region/state, service integration, identity/key policies, grants, SCP/endpoint policy, and context.
- **Follow-ups:** Alias versus key ARN? **Angle:** Trace encrypt/decrypt call. **Hook:** Identity plus key authority.

### Q050 — How do you detect unused access?
- **Testing:** Continuous least privilege. **30-second answer:** Combine access last-used data, CloudTrail, Access Analyzer, reviews, and controlled removal tests.
- **Strong answer:** Observe a representative period including rare recovery jobs; stage removal with owner, rollback, and exception expiry.
- **Follow-ups:** Data-event cost? **Angle:** Evidence window matters. **Hook:** Observe before subtracting.

## S3 — 15

### Q051 — S3 versus EBS versus EFS?
- **Testing:** Storage semantics. **30-second answer:** S3 is object/API, EBS is AZ-scoped block, EFS is regional shared NFS.
- **Strong answer:** Choose by access protocol/consistency first, then latency, throughput, sharing, durability/availability, lifecycle, backup, and cost.
- **Follow-ups:** Database on EFS? **Angle:** Workload IO pattern. **Hook:** Object, block, file.

### Q052 — Explain S3 consistency.
- **Testing:** Current behavior. **30-second answer:** S3 provides strong read-after-write consistency for object writes/deletes and listings.
- **Strong answer:** Strong storage consistency does not coordinate concurrent writers; use conditional requests/version-aware workflows for races.
- **Follow-ups:** Cross-Region replication? **Angle:** Separate local API consistency from async replication. **Hook:** Strong read, not distributed lock.

### Q053 — How do you secure an S3 bucket?
- **Testing:** Defense in depth. **30-second answer:** Block public access, least-privilege policies, controlled ownership, TLS/encryption, versioning/recovery, audit, and lifecycle.
- **Strong answer:** Include organization/endpoint conditions where suitable, KMS policy, data events by risk, Access Analyzer, retention and tested restore.
- **Follow-ups:** Public exception? **Angle:** Enumerate every access path. **Hook:** Access, protect, observe, recover.

### Q054 — SSE-S3 versus SSE-KMS?
- **Testing:** Encryption tradeoff. **30-second answer:** Both encrypt at rest; SSE-KMS adds customer-controlled key policy/audit and KMS request considerations.
- **Strong answer:** Choose from separation, cross-account access, compliance, rotation/control, cost/quota, and operational recovery—not “more secure” alone.
- **Follow-ups:** Bucket keys? **Angle:** Include KMS authorization. **Hook:** Encryption plus key control.

### Q055 — What does versioning protect?
- **Testing:** Recovery semantics. **30-second answer:** It retains versions so overwrites/deletes can be recovered; it increases retention cost and needs lifecycle.
- **Strong answer:** Define noncurrent expiration, delete-marker handling, replication, privileged permanent deletion, and restoration workflow.
- **Follow-ups:** MFA Delete? **Angle:** Restore a sampled prefix. **Hook:** Delete becomes a version.

### Q056 — S3 replication design?
- **Testing:** DR/data movement. **30-second answer:** Configure source/destination versioning, rule scope, destination/KMS permissions, metrics, and recovery/failback procedure.
- **Strong answer:** Replication is asynchronous; define RPO, ownership, delete/metadata behavior, existing-object handling, and independent protection from logical corruption.
- **Follow-ups:** RTC? **Angle:** Measure replication lag. **Hook:** Copy is async policy.

### Q057 — What is S3 Object Lock?
- **Testing:** Immutability. **30-second answer:** WORM retention/legal hold on object versions, with governance and compliance modes.
- **Strong answer:** Tie retention to legal requirements, isolate administration, understand override/delete consequences, KMS lifecycle, replication, and recovery testing.
- **Follow-ups:** Versioning? **Angle:** Test authorized and denied deletion. **Hook:** Retention on versions.

### Q058 — How do lifecycle policies work?
- **Testing:** Cost/retention. **30-second answer:** Rules transition or expire current/noncurrent objects and multipart uploads based on filters/time.
- **Strong answer:** Model minimum duration/retrieval charges, rule precedence, delete markers, replication, legal holds, and restoration lead time.
- **Follow-ups:** Immediate transition? **Angle:** Cost model with access distribution. **Hook:** Age drives class/action.

### Q059 — How do presigned URLs work?
- **Testing:** Delegated access. **30-second answer:** A signer grants time-limited permission for a specific S3 request using its own authority.
- **Strong answer:** Keep expiry short, exact method/key/content constraints, protect the URL as a bearer capability, and consider revocation via credential/policy changes.
- **Follow-ups:** Upload limits? **Angle:** Threat-model leakage. **Hook:** Temporary signed request.

### Q060 — Why does S3 return AccessDenied?
- **Testing:** Policy troubleshooting. **30-second answer:** Identify principal/action/resource then inspect identity, bucket/access-point, endpoint, organization, ownership and KMS controls.
- **Strong answer:** Use request ID/CloudTrail, distinguish bucket versus object ARN and explicit deny, correct only the missing tuple.
- **Follow-ups:** 403 versus 404? **Angle:** Avoid broad temporary allow. **Hook:** PARC plus KMS.

### Q061 — How do you optimize large uploads?
- **Testing:** Transfer design. **30-second answer:** Use multipart upload with controlled parallelism, retries/checksums, and cleanup of incomplete uploads.
- **Strong answer:** Benchmark object size, client CPU/network, endpoint/NAT, concurrency, KMS and request cost; resume failed parts idempotently.
- **Follow-ups:** Part size? **Angle:** Measure bottleneck. **Hook:** Split, parallel, verify.

### Q062 — S3 event processing semantics?
- **Testing:** Event-driven reliability. **30-second answer:** Design consumers for duplicate and out-of-order notifications using idempotency and durable state.
- **Strong answer:** Filter events, decouple through queue where appropriate, use DLQ/redrive, record object version/sequencer, and reconcile missed work.
- **Follow-ups:** Exactly once? **Angle:** Failure/replay test. **Hook:** Event is a hint; process idempotently.

### Q063 — How do you recover mass deletion?
- **Testing:** Incident recovery. **30-second answer:** Stop writers, preserve evidence, identify scope/versions, restore to controlled location, validate, then resume.
- **Strong answer:** Use versioning/Object Lock/replica/backup according to runbook, determine compromised identity, and fix destructive permissions before cutover.
- **Follow-ups:** Delete markers? **Angle:** Automate manifest-based restore. **Hook:** Contain, identify, restore, validate.

### Q064 — Access point versus bucket policy?
- **Testing:** Access at scale. **30-second answer:** Access points provide named endpoints and policies for specific applications while bucket policy remains resource-wide control.
- **Strong answer:** Use access points to separate use cases/network origins, but evaluate combined policy, Block Public Access, ownership, and KMS.
- **Follow-ups:** VPC-only? **Angle:** One policy contract per application. **Hook:** Per-use doorway.

### Q065 — How do you control S3 cost?
- **Testing:** Operational economics. **30-second answer:** Attribute storage, requests, retrieval, replication and transfer; apply lifecycle, compression/format, caching and cleanup.
- **Strong answer:** Use Storage Lens/inventory and access evidence; avoid transitions whose minimum durations/retrieval costs exceed benefit.
- **Follow-ups:** Small objects? **Angle:** Unit economics. **Hook:** Bytes, requests, movement, retention.

## EC2, Auto Scaling, and load balancing — 15

### Q066 — How do you choose an EC2 instance type?
- **Testing:** Workload fit. **30-second answer:** Measure CPU, memory, network/EBS bandwidth, architecture, local storage, latency and price under representative load.
- **Strong answer:** Right-size from percentiles and bottlenecks, test multiple families, account for burst behavior, quotas/capacity and licensing.
- **Follow-ups:** Graviton? **Angle:** Benchmark, do not guess. **Hook:** Fit the bottleneck.

### Q067 — On-Demand, Savings Plans, and Spot?
- **Testing:** Capacity economics. **30-second answer:** On-Demand is flexible; commitments discount stable use; Spot discounts interruptible spare capacity.
- **Strong answer:** Establish baseline, commit conservatively, diversify Spot and handle interruption; cost choices must preserve required capacity and recovery.
- **Follow-ups:** Reserved Instances? **Angle:** Utilization and interruption model. **Hook:** Flexible, committed, interruptible.

### Q068 — What makes an AMI production-ready?
- **Testing:** Immutable operations. **30-second answer:** Minimal, patched, scanned, versioned, reproducibly built, boot-tested and traceable with no secrets.
- **Strong answer:** Promote through environments, validate architecture/drivers/agent, deprecate old images, and roll via ASG with rollback.
- **Follow-ups:** Patch cadence? **Angle:** Launch canary. **Hook:** Bake, test, replace.

### Q069 — Instance store versus EBS?
- **Testing:** Storage durability. **30-second answer:** Instance store is host-local ephemeral high-performance storage; EBS is persistent network block storage in an AZ.
- **Strong answer:** Use instance store for reconstructible/cache/scratch data with replication; EBS for persistent volumes with snapshots and performance planning.
- **Follow-ups:** Stop behavior? **Angle:** Design for host loss. **Hook:** Ephemeral local versus persistent block.

### Q070 — Explain EBS performance dimensions.
- **Testing:** IO fundamentals. **30-second answer:** IOPS, throughput, latency, IO size, queue depth, volume and instance ceilings interact.
- **Strong answer:** Throughput roughly equals IOPS times IO size; correlate CloudWatch with `iostat`, filesystem, application waits and instance bandwidth.
- **Follow-ups:** High IOPS, slow app? **Angle:** Find limiting minimum. **Hook:** IO size connects IOPS to bytes.

### Q071 — System versus instance status check?
- **Testing:** Failure isolation. **30-second answer:** System check indicates underlying infrastructure path; instance check points to guest OS/network configuration.
- **Strong answer:** Use console/system logs, SSM/serial access, attached-volume health, recovery/replacement based on immutable design.
- **Follow-ups:** Both fail? **Angle:** Preserve evidence, restore service. **Hook:** Host side versus guest side.

### Q072 — How do you secure instance metadata?
- **Testing:** Credential protection. **30-second answer:** Require IMDSv2, limit hop/access, patch SSRF, use narrow roles, and prevent containers from reaching metadata unnecessarily.
- **Strong answer:** Metadata hardening reduces but does not replace application isolation and least privilege; monitor credential/session use.
- **Follow-ups:** Hop limit? **Angle:** Test from workload. **Hook:** Protect the credential endpoint.

### Q073 — How does target tracking scaling work?
- **Testing:** Feedback control. **30-second answer:** It adjusts capacity to keep a selected metric near a target, considering warm-up and scale-in behavior.
- **Strong answer:** Choose a demand-per-capacity metric, define min/max, startup delay, downstream limits, and test step response/oscillation.
- **Follow-ups:** CPU unsuitable when? **Angle:** Graph demand versus capacity. **Hook:** Metric must divide with scale.

### Q074 — Why might ASG fail to launch?
- **Testing:** Capacity diagnosis. **30-second answer:** Check scaling activity for quota, subnet IP, AZ capacity, launch template, AMI/KMS, IAM, or instance-type errors.
- **Strong answer:** Diversify types/AZs, keep known-good template versions, launch canaries, and alarm before maximum or IP exhaustion.
- **Follow-ups:** Spot failure? **Angle:** Read exact activity reason. **Hook:** Desired is not available.

### Q075 — ALB versus NLB?
- **Testing:** Load balancer choice. **30-second answer:** ALB offers Layer-7 HTTP routing; NLB handles Layer-4 TCP/UDP/TLS performance and static-address needs.
- **Strong answer:** Select by protocol, routing, source IP, TLS, targets, health, latency, scale, private connectivity and cost.
- **Follow-ups:** gRPC? PrivateLink? **Angle:** Start with protocol. **Hook:** L7 rules versus L4 flow.

### Q076 — What makes a good health check?
- **Testing:** Availability design. **30-second answer:** Fast, reliable readiness signal that proves a target can serve without creating a shared-dependency cascade.
- **Strong answer:** Separate startup/liveness/readiness/SLO; tune interval/threshold/grace and verify actual user paths synthetically.
- **Follow-ups:** Database check? **Angle:** Failure-mode test. **Hook:** Admit safe traffic.

### Q077 — Diagnose ALB 502/503/504.
- **Testing:** Layer mapping. **30-second answer:** 502 suggests invalid/reset target response, 503 unavailable healthy targets, 504 target response timeout.
- **Strong answer:** Confirm origin in access logs, target health, app logs/traces, protocol/port, capacity and dependency/timeout chain.
- **Follow-ups:** ALB-generated versus target code? **Angle:** Request ID and timestamps. **Hook:** Bad response, no target, slow target.

### Q078 — Rolling versus blue/green deployment?
- **Testing:** Release tradeoffs. **30-second answer:** Rolling replaces capacity incrementally; blue/green builds parallel environment and shifts traffic.
- **Strong answer:** Blue/green gives faster compute rollback but costs duplicate capacity and cannot automatically reverse data/schema side effects.
- **Follow-ups:** Canary? **Angle:** Define rollback evidence. **Hook:** Replace in place versus switch environments.

### Q079 — Why high CPU after scaling out?
- **Testing:** Bottleneck reasoning. **30-second answer:** Load may be uneven, metric lagged, state/session pinned, dependency saturated, or workload not horizontally divisible.
- **Strong answer:** Compare per-target requests/latency, connection behavior, hot keys, queues, GC, database and cache; fix cause, not only capacity.
- **Follow-ups:** Sticky sessions? **Angle:** Compare affected targets. **Hook:** More hosts do not divide every bottleneck.

### Q080 — How do you patch an EC2 fleet?
- **Testing:** Safe operations. **30-second answer:** Build/test a patched image and roll through ASG waves with health/SLO gates and rollback; use managed patching when model requires.
- **Strong answer:** Inventory urgency, canary, capacity surge, state drain, reboot, compliance evidence, exception handling, and emergency process.
- **Follow-ups:** Zero-day? **Angle:** Risk-based rings. **Hook:** Patch, prove, replace.

## EKS — 25

### Q081 — What does EKS manage?
- **Testing:** Responsibility boundary. **30-second answer:** AWS manages the Kubernetes control plane; customer owns workloads, access, data plane/Fargate configuration, add-ons, security and upgrades.
- **Strong answer:** Name control-plane availability plus customer API compatibility, nodes, CNI/CSI/DNS, RBAC, policies, telemetry and recovery.
- **Follow-ups:** etcd access? **Angle:** Shared responsibility. **Hook:** Managed control, owned platform.

### Q082 — Managed node group versus Fargate?
- **Testing:** Compute tradeoff. **30-second answer:** Node groups offer instance/daemon/storage control; Fargate removes nodes with per-pod constraints and economics.
- **Strong answer:** Compare workload compatibility, DaemonSets, privileged access, startup, isolation, bin-packing, Spot, observability and cost.
- **Follow-ups:** System pods? **Angle:** Requirements before abstraction. **Hook:** Control versus serverless pods.

### Q083 — How does VPC CNI work?
- **Testing:** Pod networking. **30-second answer:** It assigns VPC-routable addresses to pods through ENIs/prefixes on nodes.
- **Strong answer:** Plan subnet IPs, ENI/pod density, prefix delegation, security groups/network policy, warm pools and scaling failure signals.
- **Follow-ups:** IP exhaustion? **Angle:** Pod-to-ENI path. **Hook:** Pods consume network capacity.

### Q084 — Service versus Ingress?
- **Testing:** Kubernetes networking. **30-second answer:** Service provides stable internal discovery/load balancing; Ingress declares HTTP routing implemented by a controller.
- **Strong answer:** Trace DNS -> controller-created LB -> listener/rule -> Service -> ready endpoints -> pod targetPort.
- **Follow-ups:** Gateway API? **Angle:** Declaration needs controller. **Hook:** Stable service, external routes.

### Q085 — Diagnose a Pending pod.
- **Testing:** Scheduler method. **30-second answer:** Read events, then check requests, node capacity, selectors/affinity, taints, topology, quotas, PVC and IPs.
- **Strong answer:** Verify autoscaler decision, cloud capacity and pod density; change the smallest invalid constraint or add compatible capacity.
- **Follow-ups:** Preemption? **Angle:** Scheduler event is first evidence. **Hook:** Constraint meets capacity.

### Q086 — Diagnose PVC Pending.
- **Testing:** Storage troubleshooting. **30-second answer:** Inspect PVC events, StorageClass, access mode, CSI health/IAM, binding mode, topology, quota and KMS.
- **Strong answer:** `WaitForFirstConsumer` ties provisioning to pod scheduling/AZ; avoid deleting stateful objects blindly.
- **Follow-ups:** EBS multi-attach? **Angle:** Claim -> class -> CSI -> cloud volume. **Hook:** Class, controller, zone.

### Q087 — CrashLoopBackOff workflow?
- **Testing:** Pod diagnosis. **30-second answer:** Inspect events, previous logs, exit code, command/config/secrets, OOM and probes.
- **Strong answer:** Compare last good version, reproduce configuration, check dependencies and resource limits, then rollback or correct narrowly.
- **Follow-ups:** Backoff meaning? **Angle:** Previous container evidence. **Hook:** Exit before restart.

### Q088 — Readiness versus liveness versus startup probe?
- **Testing:** Health semantics. **30-second answer:** Readiness controls traffic, liveness restarts, startup delays the other probes during initialization.
- **Strong answer:** Use distinct inexpensive endpoints and thresholds; wrong liveness causes restart cascades, wrong readiness sends traffic too early.
- **Follow-ups:** Dependency checks? **Angle:** Simulate failure. **Hook:** Admit, restart, initialize.

### Q089 — Requests versus limits?
- **Testing:** Resource management. **30-second answer:** Requests drive scheduling; limits constrain runtime—CPU throttles and memory excess can OOM.
- **Strong answer:** Size from observed percentiles/load tests, account for node/system headroom, QoS and autoscaling signals.
- **Follow-ups:** No limit? **Angle:** Correlate throttling/OOM. **Hook:** Reserve versus cap.

### Q090 — What is a PodDisruptionBudget?
- **Testing:** Availability nuance. **30-second answer:** It limits voluntary concurrent pod disruptions; it does not guarantee replicas or protect involuntary failures.
- **Strong answer:** Combine with topology, capacity, replicas and graceful shutdown; impossible PDBs can block node drains/upgrades.
- **Follow-ups:** `minAvailable`? **Angle:** Test a drain. **Hook:** Voluntary disruption budget.

### Q091 — Design zero-downtime EKS upgrade.
- **Testing:** Senior operations. **30-second answer:** Preflight deprecated APIs/add-ons, upgrade control plane then add-ons/nodes in tested waves with surge, PDBs and SLO gates.
- **Strong answer:** Rehearse nonprod, canary workloads, cordon/drain, validate DNS/storage/ingress/autoscaling; blue/green for stronger rollback.
- **Follow-ups:** Roll back control plane? **Angle:** Compatibility matrix. **Hook:** API, add-ons, nodes, workloads.

### Q092 — How do you upgrade worker nodes?
- **Testing:** Disruption control. **30-second answer:** Add updated capacity, cordon/drain old nodes, reschedule safely, validate, then terminate in waves.
- **Strong answer:** Respect PDBs, local/stateful data, DaemonSets, topology, IPs, maxUnavailable, bootstrap and capacity availability.
- **Follow-ups:** Stuck drain? **Angle:** One node canary. **Hook:** Add, drain, verify, remove.

### Q093 — Cluster Autoscaler versus Karpenter?
- **Testing:** Capacity provisioning concepts. **30-second answer:** Both add nodes for unschedulable pods; Cluster Autoscaler scales predefined groups, Karpenter directly selects/provisions suitable capacity.
- **Strong answer:** Compare flexibility, consolidation, constraints, disruption, Spot handling, governance and operational maturity for current versions.
- **Follow-ups:** HPA interaction? **Angle:** Pod demand to cloud capacity. **Hook:** Groups versus direct provisioning.

### Q094 — HPA versus VPA?
- **Testing:** Scaling dimensions. **30-second answer:** HPA changes replica count; VPA recommends/changes pod requests; combined use requires avoiding signal conflicts.
- **Strong answer:** Select meaningful metrics, stabilization behavior, startup and downstream constraints; load-test scaling loops.
- **Follow-ups:** CPU utilization denominator? **Angle:** Demand, pod size, node capacity. **Hook:** More pods versus bigger pods.

### Q095 — Secure EKS workload identity.
- **Testing:** Least privilege. **30-second answer:** Assign role per service account using Pod Identity or IRSA; do not rely on broad node credentials.
- **Strong answer:** Constrain association/trust, namespace/service account, role policy and session; protect metadata and audit actual assumed sessions.
- **Follow-ups:** SDK support? **Angle:** Verify credential chain in pod. **Hook:** Pod gets its own role.

### Q096 — EKS authentication versus authorization?
- **Testing:** Access model. **30-second answer:** AWS identity authenticates to cluster; Kubernetes RBAC/access configuration authorizes API actions.
- **Strong answer:** Use access entries/current supported mechanism, federated roles, least RBAC, separated admins, audit logs and break-glass.
- **Follow-ups:** `kubectl auth can-i`? **Angle:** Test exact impersonated action. **Hook:** Enter, then authorize.

### Q097 — How do you enforce network isolation?
- **Testing:** Layered network policy. **30-second answer:** Use namespaces/account/cluster boundaries, SGs and Kubernetes network policies with a supporting data plane.
- **Strong answer:** Default-deny ingress/egress, explicitly allow DNS/dependencies, validate enforcement, observe flows, and separate high-trust workloads strongly.
- **Follow-ups:** SG for pods? **Angle:** Test real packet path. **Hook:** Policy needs enforcement.

### Q098 — Why is a Service not routing traffic?
- **Testing:** Endpoint mapping. **30-second answer:** Check selector and ready endpoints, ports/targetPort, pod listener, DNS, kube networking and policy.
- **Strong answer:** Test pod IP then ClusterIP then ingress path; compare EndpointSlices and readiness events.
- **Follow-ups:** Headless Service? **Angle:** Narrow hop by hop. **Hook:** No endpoints, no service.

### Q099 — How do you troubleshoot CoreDNS?
- **Testing:** Cluster DNS. **30-second answer:** Query from pod, inspect CoreDNS pods/logs/metrics/config/endpoints, resources, network policy and upstream Resolver.
- **Strong answer:** Compare cached/uncached and TCP/UDP, affected nodes/AZs, throttling, forwarding loops and node DNS configuration.
- **Follow-ups:** `ndots`? **Angle:** Query volume and suffix expansion. **Hook:** Client, CoreDNS, upstream.

### Q100 — How do you handle secrets in EKS?
- **Testing:** Secret lifecycle. **30-second answer:** Use workload identity to retrieve managed secrets or a controlled CSI/operator pattern; encrypt and restrict Kubernetes secrets.
- **Strong answer:** Define rotation/cache, RBAC/etcd encryption, namespace isolation, no env/log leaks, audit and failure behavior.
- **Follow-ups:** Secret refresh? **Angle:** Rotation end-to-end. **Hook:** Retrieve, restrict, rotate.

### Q101 — What causes OOMKilled?
- **Testing:** Runtime memory. **30-second answer:** Container exceeded its memory cgroup limit or node pressure led to termination/eviction; inspect status and metrics.
- **Strong answer:** Compare working set, leak/heap, request/limit, concurrency and node pressure; stabilize then profile and right-size.
- **Follow-ups:** Exit 137? **Angle:** Container versus node. **Hook:** Limit, leak, load.

### Q102 — How do you design EKS observability?
- **Testing:** Platform operations. **30-second answer:** Collect control-plane audit, Kubernetes events, node/container logs, workload RED, resource USE and traces with correlation.
- **Strong answer:** Standard labels but cap cardinality, deployment events, SLO alerts, retention/access, cost, and runbooks.
- **Follow-ups:** Missing pod logs? **Angle:** Follow request/workload/node/AZ. **Hook:** State, signal, change.

### Q103 — How do you secure the software supply chain?
- **Testing:** End-to-end security. **30-second answer:** Protect source/build, pin dependencies, scan/SBOM, sign provenance, allow trusted registries, verify at admission and deploy by digest.
- **Strong answer:** Isolate untrusted builds, short-lived roles, exception expiry, continuous runtime inventory, and tested revocation/rebuild.
- **Follow-ups:** Scan false positives? **Angle:** Risk-based policy. **Hook:** Source to running digest.

### Q104 — How do you run stateful workloads on EKS?
- **Testing:** Storage/availability. **30-second answer:** Use suitable CSI storage, StatefulSets, topology-aware provisioning, backups and application-native replication/recovery.
- **Strong answer:** Define access mode, AZ coupling, rescheduling, snapshots versus consistent backup, disruption, upgrade and restore testing.
- **Follow-ups:** EBS versus EFS? **Angle:** Failure of node/AZ/cluster. **Hook:** Stateful means data lifecycle.

### Q105 — EKS versus ECS?
- **Testing:** Platform selection. **30-second answer:** ECS is simpler AWS-native orchestration; EKS provides Kubernetes APIs/ecosystem and greater platform flexibility.
- **Strong answer:** Compare required portability/ecosystem, team skill, security model, operations, workload constraints and total cost—not feature count.
- **Follow-ups:** Fargate on both? **Angle:** Organizational fit. **Hook:** Simplicity versus Kubernetes standard.

## ECS — 10

### Q106 — Task, task definition, service, cluster?
- **Testing:** ECS model. **30-second answer:** Definition is versioned spec, task is running copy, service maintains tasks/deployments, cluster is capacity/logical grouping.
- **Strong answer:** Add capacity provider, networking, role, LB and autoscaling relationships.
- **Follow-ups:** Standalone task? **Angle:** Trace desired to running. **Hook:** Spec, copy, keeper, pool.

### Q107 — Fargate versus ECS on EC2?
- **Testing:** Compute model. **30-second answer:** Fargate removes host operations; EC2 gives instance control, broader patterns and potential steady-scale efficiency.
- **Strong answer:** Compare isolation, privileged/agent needs, scaling, startup, purchase models, utilization, patching and cost.
- **Follow-ups:** Spot? **Angle:** Total operational cost. **Hook:** Serverless tasks versus owned fleet.

### Q108 — Why does a task stop before starting?
- **Testing:** Launch troubleshooting. **30-second answer:** Inspect stopped reason for image/platform, execution role, network/DNS, secret/KMS, log driver, capacity or resource error.
- **Strong answer:** Validate exact task revision and subnet/endpoint path; correct the responsible layer and redeploy immutable revision.
- **Follow-ups:** CannotPullContainerError? **Angle:** Agent prerequisites. **Hook:** Reason before retry.

### Q109 — How do ECS deployments roll back?
- **Testing:** Release safety. **30-second answer:** Use deployment circuit breaker/alarms or blue-green controls to stop and restore a known-good task definition.
- **Strong answer:** Configure health grace, min/max healthy, spare capacity, deregistration, immutable image and database compatibility.
- **Follow-ups:** Bake time? **Angle:** Define failure threshold. **Hook:** Detect, stop, restore revision.

### Q110 — ECS service scaling design?
- **Testing:** Capacity loop. **30-second answer:** Scale task count on demand-per-task signals and ensure capacity provider/network/downstream can satisfy it.
- **Strong answer:** Use request or queue backlog metrics, warm-up/cooldown, min/max, scheduled baseline and downstream protection.
- **Follow-ups:** Capacity provider strategy? **Angle:** Service and host capacity separately. **Hook:** Tasks plus capacity.

### Q111 — ECS networking modes?
- **Testing:** Connectivity. **30-second answer:** `awsvpc` gives tasks ENIs and is required for Fargate; other EC2 modes share/map host networking differently.
- **Strong answer:** `awsvpc` improves identity/isolation but consumes IP/ENI capacity; validate target type, SG and port mapping.
- **Follow-ups:** IP target? **Angle:** Trace task ENI. **Hook:** Task gets network identity.

### Q112 — Why is ECS target unhealthy?
- **Testing:** LB integration. **30-second answer:** Check container port/listener, target type, SG, health path/status, bind address, startup and application logs.
- **Strong answer:** Compare container health, ECS task health and target-group health; they are distinct signals.
- **Follow-ups:** Grace period? **Angle:** Directly test target. **Hook:** Running is not ready.

### Q113 — How do tasks retrieve secrets?
- **Testing:** Identity separation. **30-second answer:** ECS can inject referenced Secrets Manager/Parameter Store values using execution-role permissions, or app retrieves at runtime using task role.
- **Strong answer:** Choose based on rotation/refresh, exposure, SDK behavior, least privilege, KMS and deployment restart semantics.
- **Follow-ups:** Rotation without restart? **Angle:** Follow credential lifecycle. **Hook:** Agent injects; app retrieves.

### Q114 — What are capacity providers?
- **Testing:** ECS capacity strategy. **30-second answer:** They connect task placement to Fargate or Auto Scaling capacity with base and weighted strategies.
- **Strong answer:** Mix stable and Spot capacity according to interruption tolerance, scaling lag, AZ/type diversity and service criticality.
- **Follow-ups:** Managed scaling? **Angle:** Desired tasks versus instances. **Hook:** Placement chooses capacity source.

### Q115 — How do you observe ECS?
- **Testing:** Operability. **30-second answer:** Correlate service events, task stopped reasons, container logs, LB health/access, utilization, deployment and traces.
- **Strong answer:** Add Container Insights where justified, request IDs, version/task dimensions, SLO alerts and logs for execution/runtime identity failures.
- **Follow-ups:** High cardinality? **Angle:** Follow user request to task/dependency. **Hook:** Event, task, target, app.

## RDS and Aurora — 15

### Q116 — Multi-AZ versus read replica?
- **Testing:** HA versus scale. **30-second answer:** Multi-AZ is primarily availability/failover; read replicas asynchronously scale reads and may support DR patterns.
- **Strong answer:** Engine/deployment type matters; define replication, endpoint, consistency, promotion, lag and recovery behavior precisely.
- **Follow-ups:** Can standby serve reads? **Angle:** Name requirement first. **Hook:** HA versus read scale.

### Q117 — RDS versus Aurora?
- **Testing:** Database selection. **30-second answer:** RDS manages standard engines; Aurora is AWS-designed MySQL/PostgreSQL-compatible architecture with distributed storage and cluster features.
- **Strong answer:** Compare compatibility, performance evidence, availability/failover, replicas/global needs, features, migration, skills and total cost.
- **Follow-ups:** Serverless? **Angle:** Benchmark workload. **Hook:** Compatibility plus architecture.

### Q118 — How do automated backups and PITR work conceptually?
- **Testing:** Recovery. **30-second answer:** RDS retains backups and transaction logs within a window to restore a new database near a chosen time.
- **Strong answer:** Define retention, KMS, Region/account copies, restore duration, parameter/network configuration, validation and cutover.
- **Follow-ups:** Does restore overwrite? **Angle:** Time the full restore. **Hook:** Restore creates new.

### Q119 — How do you troubleshoot a slow database?
- **Testing:** Evidence method. **30-second answer:** Start with transaction SLO, waits/top SQL/locks/connections, then CPU/memory/cache/storage and changes.
- **Strong answer:** Inspect execution plan, rows/index/statistics, temp spill, pool and dependency; stabilize safely and test correction on representative data.
- **Follow-ups:** High CPU? **Angle:** Waits before resizing. **Hook:** Query, wait, resource, change.

### Q120 — What causes replica lag?
- **Testing:** Replication mechanics. **30-second answer:** Heavy writes, long transactions/DDL, insufficient replica compute/IO, blocked apply, network or heavy replica reads.
- **Strong answer:** Measure engine-specific lag and apply state, protect consistency-sensitive reads, scale/remove blocker, and set lag routing thresholds.
- **Follow-ups:** Promotion risk? **Angle:** Compare generation versus apply rate. **Hook:** Replica cannot apply fast enough.

### Q121 — How do you design connection pooling?
- **Testing:** Database protection. **30-second answer:** Bound connections per app instance based on DB capacity, reuse them, time out leaks and prevent retry storms.
- **Strong answer:** Model total fleet maximum, transaction duration, server resources, failover refresh, RDS Proxy suitability and observability.
- **Follow-ups:** Scale-out danger? **Angle:** Pool budget is global. **Hook:** App replicas multiply connections.

### Q122 — What is RDS Proxy for?
- **Testing:** Managed connection proxy. **30-second answer:** It pools/manages database connections to absorb churn and improve application failover behavior for supported use cases.
- **Strong answer:** It does not fix slow SQL or unlimited transactions; evaluate pinning, auth, latency, cost, engine support and pool configuration.
- **Follow-ups:** Session state? **Angle:** Measure before/after connections. **Hook:** Smooth connections, not queries.

### Q123 — How do you perform a safe schema migration?
- **Testing:** Deployment/data compatibility. **30-second answer:** Use expand/contract: add compatible schema, deploy dual-compatible code, migrate/validate, switch, then remove later.
- **Strong answer:** Assess locks/table rewrite, throttle/backfill, observability, cancellation/rollback and replication impact on production-like data.
- **Follow-ups:** Destructive rollback? **Angle:** Separate code and data reversibility. **Hook:** Expand, migrate, contract.

### Q124 — How do you encrypt and rotate DB credentials?
- **Testing:** Secret lifecycle. **30-second answer:** Store in Secrets Manager, authorize workload identity, rotate with overlapping stages, refresh pools and audit access.
- **Strong answer:** Separate admin/app users, TLS, KMS policy, failure rollback, client caching and end-to-end rotation test.
- **Follow-ups:** IAM DB auth? **Angle:** Credential works after rotation. **Hook:** Rotate value and consumers.

### Q125 — RDS storage full: response?
- **Testing:** Safe incident response. **30-second answer:** Stop growth/load safely, identify logs/temp/table/index source, increase storage if appropriate, then correct retention/query/capacity.
- **Strong answer:** Watch autoscaling limits, free storage, transaction logs and engine behavior; avoid risky cleanup without backup and ownership.
- **Follow-ups:** Can shrink? **Angle:** Growth rate and time to exhaustion. **Hook:** Stabilize space, find growth.

### Q126 — How do you test failover?
- **Testing:** Resilience proof. **30-second answer:** In a controlled window trigger supported failover and measure detection, DNS/pool reconnect, transaction behavior and user RTO.
- **Strong answer:** Define expected errors, bounded retries/idempotency, monitoring, stakeholders, abort/failback, and corrective actions.
- **Follow-ups:** Data loss? **Angle:** Test application, not DB event only. **Hook:** Failover ends when users recover.

### Q127 — Snapshot copy across accounts/Regions?
- **Testing:** DR/security. **30-second answer:** Share/copy snapshots with compatible KMS permissions and destination-owned encryption, then configure network/parameters on restore.
- **Strong answer:** Automate retention and copy status, isolate recovery account, test engine/version/option dependencies and restore time.
- **Follow-ups:** AWS-managed key? **Angle:** Key lifecycle is recovery dependency. **Hook:** Data copy plus key access.

### Q128 — How do you monitor RDS?
- **Testing:** Database observability. **30-second answer:** Track transaction SLO, connections, CPU/memory, storage/IO, replica lag, waits/top SQL, logs and events.
- **Strong answer:** Use engine-aware tooling/Performance Insights capabilities, Enhanced Monitoring where justified, deployment correlation and owned thresholds.
- **Follow-ups:** Average latency? **Angle:** Tail and workload class. **Hook:** User, query, resource, replication.

### Q129 — When use Aurora Global Database?
- **Testing:** Multi-Region data design. **30-second answer:** For cross-Region read locality and low-RPO DR where Aurora compatibility, cost and promotion model fit.
- **Strong answer:** Define writer Region, replication lag, planned/unplanned failover, client endpoints, write forwarding/consistency, quotas and failback reconciliation.
- **Follow-ups:** Active/active writes? **Angle:** Write ownership first. **Hook:** Global reads, controlled writer.

### Q130 — How do you reduce database cost safely?
- **Testing:** Optimization without risk. **30-second answer:** Right-size from utilization/waits, optimize queries/storage/connections, schedule nonprod, use commitments after stable baseline.
- **Strong answer:** Include replicas, backup retention, I/O, transfer and licenses; load/failover-test changes and preserve headroom/SLO.
- **Follow-ups:** Downsize risk? **Angle:** Cost per transaction with resilience. **Hook:** Optimize work before capacity.

## CI/CD, Terraform, GitHub Actions, and Jenkins — 20

### Q131 — What makes a production CI/CD pipeline?
- **Testing:** Delivery architecture. **30-second answer:** Reviewed source builds one immutable artifact, proves controls, promotes with least privilege, verifies SLOs and can recover.
- **Strong answer:** Include provenance/SBOM, test/policy gates, environment approvals, concurrency, canary/blue-green, database compatibility and audit evidence.
- **Follow-ups:** Speed versus control? **Angle:** Automate risk-proportional gates. **Hook:** Build once, promote, prove.

### Q132 — Why build once and promote?
- **Testing:** Artifact integrity. **30-second answer:** It ensures production receives the exact artifact tested, avoiding environment-specific rebuild drift.
- **Strong answer:** Identify by immutable digest, attach provenance/signature, separate config, restrict registry promotion, and record deployment mapping.
- **Follow-ups:** Environment config? **Angle:** Verify running digest. **Hook:** Same bits, different config.

### Q133 — Canary versus blue/green?
- **Testing:** Release tradeoff. **30-second answer:** Canary sends a small traffic share to new version; blue/green prepares parallel environments and switches traffic.
- **Strong answer:** Canary needs representative traffic/metrics; blue/green needs duplicate capacity. Both require data compatibility and explicit promotion/rollback.
- **Follow-ups:** Sticky sessions? **Angle:** Define success metrics. **Hook:** Sample traffic versus switch environment.

### Q134 — How do you secure pipeline credentials?
- **Testing:** Supply-chain security. **30-second answer:** Prefer OIDC/short-lived roles, least privilege per stage/environment, protected approvals and no secret output/artifacts.
- **Strong answer:** Isolate untrusted builds, rotate remaining secrets, pin actions/plugins, audit sessions and limit production network/data paths.
- **Follow-ups:** Self-hosted runner? **Angle:** Threat-model PR code. **Hook:** Federate, scope, isolate.

### Q135 — What is Terraform state?
- **Testing:** Core Terraform. **30-second answer:** It maps configuration resource addresses to remote objects and metadata used for planning; it can contain sensitive values.
- **Strong answer:** Protect remote state with encryption, versioning, locking, least privilege, backups and one-writer controls; separate blast-radius states.
- **Follow-ups:** Is it a cache? **Angle:** Losing state changes ownership mapping. **Hook:** Map, not mere cache.

### Q136 — How do you configure Terraform locking now?
- **Testing:** Current backend knowledge. **30-second answer:** For supported Terraform versions the S3 backend can use native lockfiles with `use_lockfile = true`; DynamoDB locking is deprecated.
- **Strong answer:** Also enable bucket versioning/encryption and CI concurrency; verify version-specific official docs and migrate locking deliberately.
- **Follow-ups:** Force unlock? **Angle:** Prove no active writer. **Hook:** Lock plus versioned recovery.

### Q137 — Handle Terraform partial apply.
- **Testing:** Recovery discipline. **30-second answer:** Freeze writers, preserve logs/state, compare cloud reality and fresh plan, fix cause, review and reconcile.
- **Strong answer:** Successful changes are recorded; do not rerun blindly or edit state. Use import/state operations only with backup and exact ownership proof.
- **Follow-ups:** Rollback? **Angle:** Desired convergence versus reversible change. **Hook:** Freeze, inspect, re-plan.

### Q138 — Recover corrupted Terraform state.
- **Testing:** High-risk operations. **30-second answer:** Stop applies, preserve versions, validate backend lineage/serial and real resources, restore known-good state or repair precisely.
- **Strong answer:** Use S3 versioning, refresh-only/normal plans cautiously, peer review, and fix locking/access root cause before apply.
- **Follow-ups:** Manual edit? **Angle:** Never guess resource mapping. **Hook:** Preserve, compare, restore, prove.

### Q139 — How do you manage drift?
- **Testing:** IaC governance. **30-second answer:** Detect scheduled plans, determine whether change is authorized, then encode/import it or let reviewed code restore state.
- **Strong answer:** Restrict console mutation, record emergency changes, avoid automatic destructive reconciliation, and measure recurring drift causes.
- **Follow-ups:** `ignore_changes`? **Angle:** Intent before convergence. **Hook:** Detect, decide, reconcile.

### Q140 — Refactor Terraform without recreation.
- **Testing:** State/address safety. **30-second answer:** Use `moved` blocks for address changes and import for existing objects; review plan until no unintended replacement.
- **Strong answer:** Pin versions, migrate in small steps, back up state, understand `for_each` keys and force-new attributes.
- **Follow-ups:** Module rename? **Angle:** Address mapping proof. **Hook:** Move address, not resource.

### Q141 — How do you design Terraform modules?
- **Testing:** Maintainability. **30-second answer:** Cohesive opinionated modules with typed/validated inputs, useful outputs, versioning, documentation, examples and tests.
- **Strong answer:** Avoid mega-modules and exposing provider internals; define ownership, compatibility and upgrade path.
- **Follow-ups:** Composition? **Angle:** Stable contract. **Hook:** Small interface, clear lifecycle.

### Q142 — Why is `sensitive = true` insufficient?
- **Testing:** Secret handling. **30-second answer:** It redacts selected CLI display but values can remain in state and plans.
- **Strong answer:** Pass references instead of secret values, retrieve at runtime, restrict state/artifacts and avoid secret-generating resources where possible.
- **Follow-ups:** Environment variables? **Angle:** Trace every persistence location. **Hook:** Redacted is not absent.

### Q143 — GitHub Actions OIDC flow?
- **Testing:** Federation. **30-second answer:** Workflow obtains GitHub OIDC token; AWS trust validates claims; STS returns temporary role credentials.
- **Strong answer:** Grant `id-token: write`, constrain audience/subject/repository/environment, minimize role policy/session and audit CloudTrail.
- **Follow-ups:** Branch condition? **Angle:** Inspect claims safely. **Hook:** Token exchange, temporary role.

### Q144 — `pull_request` versus `pull_request_target` risk?
- **Testing:** Workflow security. **30-second answer:** `pull_request_target` runs in privileged base context; executing untrusted PR code there can expose authority.
- **Strong answer:** Use unprivileged checks for fork code, never checkout/execute attacker content with secrets/write token, and gate trusted follow-on workflows.
- **Follow-ups:** Artifact poisoning? **Angle:** Trust boundary per event. **Hook:** Target context is privileged.

### Q145 — Reusable workflow versus composite action?
- **Testing:** GitHub Actions design. **30-second answer:** Reusable workflows share jobs/governance; composite actions bundle steps within a caller job.
- **Strong answer:** Define explicit inputs/secrets/permissions, pin versions and use reusable workflows for central deployment policy.
- **Follow-ups:** Nested permissions? **Angle:** Least privilege contract. **Hook:** Jobs versus steps.

### Q146 — How do you prevent concurrent deployments?
- **Testing:** Serialization. **30-second answer:** Use environment concurrency/locks so only one production mutation runs, with non-canceling semantics for critical applies.
- **Strong answer:** Coordinate pipeline deployment lock with Terraform state locking; handle stale runs, approvals, timeouts and idempotent resume.
- **Follow-ups:** Multi-repository? **Angle:** One authoritative deployment queue. **Hook:** Two locks, one writer.

### Q147 — Jenkins controller versus agent?
- **Testing:** Jenkins architecture. **30-second answer:** Controller orchestrates/configures; agents execute builds, preferably ephemeral and isolated.
- **Strong answer:** Keep builds off controller, manage configuration/plugins as code, patch, back up/restore, constrain credentials and untrusted jobs.
- **Follow-ups:** Executors on controller? **Angle:** Blast radius. **Hook:** Controller schedules; agent works.

### Q148 — Jenkins queue is stuck; troubleshoot.
- **Testing:** Operational method. **30-second answer:** Read queue reason, then agent labels/capacity/status, cloud provisioning/quota, executor/locks and controller health.
- **Strong answer:** Distinguish demand spike from label mismatch, dead agent, stale lock or controller/plugin issue; restore capacity safely and alarm on queue time.
- **Follow-ups:** Agent flapping? **Angle:** Queue item to required label. **Hook:** Demand, match, provision, execute.

### Q149 — Rollback versus roll-forward?
- **Testing:** Incident decision. **30-second answer:** Choose the fastest safe restoration considering change reversibility, data/schema compatibility, confidence and elapsed impact.
- **Strong answer:** Predefine triggers; immutable compute often rolls back, while irreversible data change may require forward fix/restore/reconciliation.
- **Follow-ups:** Who decides? **Angle:** Time-box and evidence. **Hook:** Restore safely, not ideologically.

### Q150 — How do you discuss limited production experience?
- **Testing:** Integrity and transfer of knowledge. **30-second answer:** State the boundary clearly, describe hands-on lab/training work, then explain production-safe design, validation and rollback.
- **Strong answer:** “I have not personally owned this failure in production. I reproduced the workflow in a lab; in production I would confirm scope, use read-only evidence, follow change control, stage a reversible mitigation, and involve the service owner.”
- **Follow-ups:** What did you actually do? **Angle:** Name concrete commands/artifacts without inventing outcomes. **Hook:** Boundary, evidence, approach.

## Practice rule

A senior answer usually follows: requirement -> mechanism -> tradeoff -> failure mode -> evidence -> safe change -> prevention. If experience is lab-based, say so once and continue with technically sound reasoning.
