# 21 — AWS DevOps Interview Questions

Exactly 150 high-value questions. Answer aloud before reading. “Strong answer” is a compact outline to expand with assumptions and tradeoffs; it is not a script to memorize.

## VPC and networking — 30

### Q001 — What makes a subnet public?
- **What interviewer is testing:** Routing fundamentals. **Short answer:** Its route table has a route to an internet gateway; resources also need public addressing and permitted traffic.
- **Strong answer:** Separate subnet routing from resource reachability: IGW attachment/route, public IPv4 or IPv6 path, SG/NACL, listener, and return path all matter.
- **Follow-up questions:** Can private IP reach internet? Does SG allow deny? **Troubleshooting / architecture angle:** Trace one packet. **Memory hook:** Public is a route, not a label.

### Q002 — Security group versus NACL?
- **What interviewer is testing:** Stateful versus stateless control. **Short answer:** SG is stateful allow-only at ENI; NACL is ordered stateless allow/deny at subnet.
- **Strong answer:** Use SGs for workload least privilege and NACLs as coarse guardrails; NACLs need explicit return/ephemeral-port rules.
- **Follow-up questions:** Rule evaluation? Return traffic? **Troubleshooting / architecture angle:** Check both directions. **Memory hook:** SG remembers; NACL forgets.

### Q003 — How does route selection work?
- **What interviewer is testing:** Packet routing. **Short answer:** Most-specific matching destination prefix wins; static/propagated precedence then matters for equal prefixes.
- **Strong answer:** Identify source subnet table, destination, longest prefix, target health, intermediate route domains, and symmetric return path.
- **Follow-up questions:** Local route? Blackhole? **Troubleshooting / architecture angle:** Forward and return tables. **Memory hook:** Most specific first.

### Q004 — NAT gateway versus internet gateway?
- **What interviewer is testing:** Egress design. **Short answer:** IGW enables internet routing for publicly addressed resources; NAT translates private IPv4 outbound flows and rejects unsolicited inbound initiation.
- **Strong answer:** Place zonal NAT per AZ for resilience, route private subnets locally, monitor ports/errors, and use endpoints to reduce NAT dependency/cost.
- **Follow-up questions:** IPv6? HA? **Troubleshooting / architecture angle:** Trace address translation. **Memory hook:** IGW routes; NAT translates.

### Q005 — Why deploy one NAT gateway per AZ?
- **What interviewer is testing:** Zonal resilience/cost. **Short answer:** It removes cross-AZ egress dependency and can reduce cross-AZ data transfer.
- **Strong answer:** Each private subnet routes to its AZ’s NAT; weigh availability and traffic cost against extra hourly cost.
- **Follow-up questions:** Failure behavior? Alternatives? **Troubleshooting / architecture angle:** Inspect subnet route ownership. **Memory hook:** Keep zonal traffic zonal.

### Q006 — Gateway versus interface endpoint?
- **What interviewer is testing:** Private service access. **Short answer:** Gateway endpoints are route targets for S3/DynamoDB; interface endpoints are PrivateLink ENIs for supported services.
- **Strong answer:** Compare DNS, SG, endpoint policy, subnet/AZ placement, service support, hourly and data cost.
- **Follow-up questions:** Does endpoint grant access? **Troubleshooting / architecture angle:** Network and authorization separately. **Memory hook:** Route target versus ENI.

### Q007 — Peering versus Transit Gateway?
- **What interviewer is testing:** Network topology. **Short answer:** Peering is simple non-transitive point-to-point; TGW is scalable hub-and-spoke with route domains.
- **Strong answer:** Select on connection count, segmentation, inspection, hybrid routing, throughput/cost, and operational ownership; neither supports overlapping CIDRs directly.
- **Follow-up questions:** Transitivity? Appliance mode? **Troubleshooting / architecture angle:** Draw route domains. **Memory hook:** Pair versus hub.

### Q008 — PrivateLink versus peering?
- **What interviewer is testing:** Service versus network exposure. **Short answer:** PrivateLink exposes a service privately; peering connects VPC address spaces bidirectionally through routes.
- **Strong answer:** PrivateLink reduces lateral reach and CIDR coupling but adds endpoint/service cost and producer design; use it for consumer-to-service access.
- **Follow-up questions:** DNS? Initiation direction? **Troubleshooting / architecture angle:** Define what must be reachable. **Memory hook:** Service, not subnet.

### Q009 — Direct Connect versus VPN?
- **What interviewer is testing:** Hybrid tradeoffs. **Short answer:** DX offers private predictable connectivity; VPN is encrypted over internet and faster to establish.
- **Strong answer:** Use redundant DX locations/connections and often VPN backup; address BGP, encryption needs, bandwidth, lead time, and failure testing.
- **Follow-up questions:** Is DX encrypted? **Troubleshooting / architecture angle:** Test each redundant path. **Memory hook:** Predictability versus speed.

### Q010 — How do you troubleshoot a timeout between EC2 instances?
- **What interviewer is testing:** Layered diagnosis. **Short answer:** Resolve DNS, verify source/destination/port, routes both ways, SG, NACL, listener, host firewall, then app.
- **Strong answer:** Use Reachability Analyzer for modeled path, Flow Logs for five-tuple evidence, and packet capture where controlled; correlate exact timestamps.
- **Follow-up questions:** Refused versus timeout? **Troubleshooting / architecture angle:** RSPAR workflow. **Memory hook:** Resolve, source, path, allow, return.

### Q011 — What do VPC Flow Logs prove?
- **What interviewer is testing:** Evidence limits. **Short answer:** They record flow metadata and accept/reject status at selected interfaces; not payload or application health.
- **Strong answer:** Match five-tuple/time/action, account for aggregation and logging configuration, then combine with route/config and application logs.
- **Follow-up questions:** Can they show DNS content? **Troubleshooting / architecture angle:** Know signal boundaries. **Memory hook:** Flow, not packet body.

### Q012 — What does Reachability Analyzer do?
- **What interviewer is testing:** Tool accuracy. **Short answer:** It statically analyzes AWS network configuration for a modeled path; it does not send traffic.
- **Strong answer:** Use it to identify route/security/config blockers, then verify DNS, runtime listener, host policy, and application separately.
- **Follow-up questions:** Intermittent failures? **Troubleshooting / architecture angle:** Configuration proof, runtime proof. **Memory hook:** Model, not probe.

### Q013 — How do ephemeral ports affect NACLs?
- **What interviewer is testing:** Stateless return path. **Short answer:** Return traffic commonly uses client ephemeral ports, which stateless NACLs must explicitly permit.
- **Strong answer:** Identify who initiates, protocol/port, OS/load-balancer port behavior, and define both ingress and egress rules without overly broad exposure.
- **Follow-up questions:** SG behavior? **Troubleshooting / architecture angle:** Reverse the five-tuple. **Memory hook:** NACL needs the reply rule.

### Q014 — What is asymmetric routing risk?
- **What interviewer is testing:** Stateful path awareness. **Short answer:** Request and response traverse different stateful devices, so return traffic may lack connection state and be dropped.
- **Strong answer:** Inspect TGW/firewall/NAT route symmetry, appliance mode, and propagation; correlate both directions in logs.
- **Follow-up questions:** How fix? **Troubleshooting / architecture angle:** Draw forward and return hops. **Memory hook:** State must see both ways.

### Q015 — How do you plan VPC CIDRs?
- **What interviewer is testing:** Scale planning. **Short answer:** Allocate non-overlapping ranges with growth for subnets, AZs, pods, endpoints, and hybrid networks using IPAM.
- **Strong answer:** Forecast ENI-heavy services, reserve expansion, segment route domains, document ownership, and prevent overlaps through organization controls.
- **Follow-up questions:** Exhaustion recovery? **Troubleshooting / architecture angle:** Measure actual IP consumers. **Memory hook:** Addresses are capacity.

### Q016 — How do you handle subnet IP exhaustion?
- **What interviewer is testing:** Production recovery. **Short answer:** Find ENI consumers, reclaim leaks, add secondary VPC CIDR/new subnets, and migrate capacity safely.
- **Strong answer:** Alarm on available IPs; for EKS evaluate pod networking/prefix delegation; never assume a subnet can be resized in place.
- **Follow-up questions:** Which services consume ENIs? **Troubleshooting / architecture angle:** Inventory before change. **Memory hook:** Add subnet, not resize.

### Q017 — How does DNS work in a VPC?
- **What interviewer is testing:** Resolver architecture. **Short answer:** Workloads normally query Route 53 Resolver; VPC attributes, private zones, Resolver rules/endpoints, and caches shape answers.
- **Strong answer:** Separate resolution from connectivity; inspect zone association, split-horizon overlap, DHCP settings, forwarding loops, TTL and app caching.
- **Follow-up questions:** Hybrid names? **Troubleshooting / architecture angle:** Query authoritative and recursive layers. **Memory hook:** Answer first, path second.

### Q018 — Private hosted zone problem diagnosis?
- **What interviewer is testing:** DNS troubleshooting. **Short answer:** Verify VPC association, exact zone/record, Resolver path, overlapping zones, record type, and client cache.
- **Strong answer:** Query from affected VPC and a known-good resolver, inspect forwarding rules and zone precedence, then test target independently.
- **Follow-up questions:** NXDOMAIN caching? **Troubleshooting / architecture angle:** Capture resolver and TTL. **Memory hook:** Zone association before route.

### Q019 — How would you centralize egress?
- **What interviewer is testing:** Inspection architecture. **Short answer:** Route spoke egress through controlled TGW route domains to inspection/NAT, while preserving symmetry and zonal resilience.
- **Strong answer:** Compare decentralized NAT/endpoints versus centralized inspection on cost, latency, blast radius, scaling, route complexity, and ownership.
- **Follow-up questions:** Cross-AZ cost? **Troubleshooting / architecture angle:** Failure-domain diagram. **Memory hook:** Central control, central dependency.

### Q020 — How do you segment a multi-account network?
- **What interviewer is testing:** Blast radius. **Short answer:** Separate VPCs/accounts, attach through TGW with distinct route tables, and expose shared services explicitly.
- **Strong answer:** No broad transitive defaults; isolate prod/nonprod/security zones, control propagation, centralize DNS selectively, and validate paths continuously.
- **Follow-up questions:** Shared services risk? **Troubleshooting / architecture angle:** Route domains are trust domains. **Memory hook:** Attach does not mean route.

### Q021 — IPv4 versus IPv6 egress?
- **What interviewer is testing:** Dual-stack understanding. **Short answer:** IPv4 private egress often uses NAT; IPv6 uses globally unique addresses with an egress-only IGW for outbound-only initiation.
- **Strong answer:** Update routes, SG/NACL, DNS, logging, application binding, and third-party allowlists for both protocols; dual stack doubles paths to test.
- **Follow-up questions:** NAT64? **Troubleshooting / architecture angle:** Test A and AAAA independently. **Memory hook:** IPv6 is not NATed IPv4.

### Q022 — How do you restrict S3 access to a VPC endpoint?
- **What interviewer is testing:** Layered policy. **Short answer:** Use bucket policy conditions plus endpoint and identity policies, with careful exceptions for trusted administration/services.
- **Strong answer:** Test exact principal/action/resource/source endpoint and account for KMS; a restrictive bucket condition can break AWS service workflows.
- **Follow-up questions:** Console access? **Troubleshooting / architecture angle:** Simulate every access path. **Memory hook:** Path condition plus identity.

### Q023 — Why can ping fail while HTTPS works?
- **What interviewer is testing:** Protocol distinction. **Short answer:** ICMP may be blocked or unsupported while TCP 443 is routed, allowed, and served.
- **Strong answer:** Test the actual application protocol; security controls and load balancers handle protocols differently.
- **Follow-up questions:** Better tools? **Troubleshooting / architecture angle:** `curl`, TCP connect, traces. **Memory hook:** Test what users use.

### Q024 — What is a bastion alternative?
- **What interviewer is testing:** Secure operations. **Short answer:** Systems Manager Session Manager provides audited access without public IPs or inbound SSH when prerequisites are met.
- **Strong answer:** Use federated identity, narrow session permissions, logging/encryption, private endpoints/egress, patched agent, and break-glass procedure.
- **Follow-up questions:** If SSM fails? **Troubleshooting / architecture angle:** Verify agent/IAM/endpoints. **Memory hook:** Identity-mediated access.

### Q025 — How would you debug intermittent DNS latency?
- **What interviewer is testing:** Evidence correlation. **Short answer:** Measure client, cache, Resolver, CoreDNS if present, forwarding endpoints, upstream servers, and network loss by timestamp.
- **Strong answer:** Compare cached/uncached queries, UDP/TCP fallback, saturation, forwarding loops, TTL behavior, and affected AZs.
- **Follow-up questions:** Large response? **Troubleshooting / architecture angle:** Preserve query name/type/server. **Memory hook:** Cache, resolver, authority.

### Q026 — What is split-horizon DNS?
- **What interviewer is testing:** Naming design. **Short answer:** The same name returns different answers based on resolver/network context, often public versus private zones.
- **Strong answer:** Use deliberately; document zone association and hybrid forwarding, avoid surprising overlap, and test from each resolution domain.
- **Follow-up questions:** Failure modes? **Troubleshooting / architecture angle:** Compare answers from contexts. **Memory hook:** Same name, different view.

### Q027 — How do you make hybrid DNS resilient?
- **What interviewer is testing:** Enterprise integration. **Short answer:** Deploy Resolver endpoints across AZs, redundant on-prem resolvers, scoped forwarding rules, monitored capacity, and loop-free fallback.
- **Strong answer:** Define namespace authority, TCP/UDP 53 paths, security controls, query logging, failure behavior, and test loss of each endpoint.
- **Follow-up questions:** Conditional forwarding? **Troubleshooting / architecture angle:** Draw authority chain. **Memory hook:** Know who owns each suffix.

### Q028 — What causes NAT data cost surprises?
- **What interviewer is testing:** Cost reasoning. **Short answer:** High internet/service traffic through NAT, cross-AZ paths, and duplicated transfer can dominate.
- **Strong answer:** Attribute bytes by flow/service, add endpoints where economics/security justify, keep same-AZ routing, cache or redesign chatty calls.
- **Follow-up questions:** Endpoint costs? **Troubleshooting / architecture angle:** Compare total cost, not hourly only. **Memory hook:** Follow the bytes.

### Q029 — Design ingress for private workloads.
- **What interviewer is testing:** Layered architecture. **Short answer:** Public ALB/NLB spans public subnets while targets run privately; SG references constrain LB-to-target traffic.
- **Strong answer:** Add WAF/CloudFront as required, TLS/certificates, health checks, zonal capacity, access logs, egress/dependency paths, and DDoS posture.
- **Follow-up questions:** NLB SG/source IP? **Troubleshooting / architecture angle:** Draw listener-to-target. **Memory hook:** Public front, private target.

### Q030 — Your network change caused outage; what now?
- **What interviewer is testing:** Incident discipline. **Short answer:** Freeze changes, define impact, roll back the known change if safe, preserve flow/config evidence, and verify restoration.
- **Strong answer:** Compare before/after routes/policies, check both directions, communicate, then root-cause and add IaC plan/reachability tests and staged rollout.
- **Follow-up questions:** When not rollback? **Troubleshooting / architecture angle:** Smallest reversible mitigation. **Memory hook:** Stabilize, prove, prevent.

## IAM — 20

### Q031 — Explain IAM policy evaluation.
- **What interviewer is testing:** Authorization fundamentals. **Short answer:** Applicable allows grant unless an explicit deny in any relevant policy plane overrides; otherwise implicit deny.
- **Strong answer:** Evaluate principal, action, resource, context across identity/resource policy, boundary, session, SCP/RCP, endpoint and service-specific policies.
- **Follow-up questions:** Cross-account? **Troubleshooting / architecture angle:** Trace exact request. **Memory hook:** Deny beats allow.

### Q032 — Role versus user?
- **What interviewer is testing:** Identity design. **Short answer:** A user is a long-lived identity; a role is assumed to obtain temporary session credentials.
- **Strong answer:** Prefer federation and roles for humans/workloads, narrow trust and permissions, short sessions, and avoid static keys.
- **Follow-up questions:** Instance profile? **Troubleshooting / architecture angle:** Identify credential source. **Memory hook:** Roles are sessions.

### Q033 — Trust policy versus permissions policy?
- **What interviewer is testing:** AssumeRole model. **Short answer:** Trust says who may assume; permissions say what the resulting role session may do.
- **Strong answer:** Caller permission, role trust conditions, session policy, boundary and SCP can all affect the result.
- **Follow-up questions:** Resource-based delegation? **Troubleshooting / architecture angle:** Separate entry from authority. **Memory hook:** Who enters; what they do.

### Q034 — What is a permissions boundary?
- **What interviewer is testing:** Delegation guardrail. **Short answer:** It caps permissions identity policies can grant; it grants no permissions itself.
- **Strong answer:** Use it for delegated role creation while recognizing resource policies/session semantics and explicit denies need full evaluation.
- **Follow-up questions:** Boundary versus SCP? **Troubleshooting / architecture angle:** Intersection of effective permissions. **Memory hook:** Ceiling, not grant.

### Q035 — What is an SCP?
- **What interviewer is testing:** Organization governance. **Short answer:** It limits maximum permissions in member accounts; it does not grant access.
- **Strong answer:** Apply carefully by OU, test service-linked/emergency paths, and diagnose explicit organizational denies from the actual account/session.
- **Follow-up questions:** Management account? **Troubleshooting / architecture angle:** Guardrail blast radius. **Memory hook:** Org ceiling.

### Q036 — How do you achieve least privilege?
- **What interviewer is testing:** Practical governance. **Short answer:** Start from required actions/resources/conditions, use short-lived roles, observe actual use, then continuously refine.
- **Strong answer:** Split runtime/deploy/admin roles; use Access Analyzer, CloudTrail, policy simulation, access last-used data, review and expiry.
- **Follow-up questions:** New workload bootstrap? **Troubleshooting / architecture angle:** Test business operations. **Memory hook:** Need, scope, observe, shrink.

### Q037 — How do you troubleshoot AccessDenied?
- **What interviewer is testing:** Systematic method. **Short answer:** Capture session principal, action, resource, context/request ID, then evaluate every applicable policy plane and conditions.
- **Strong answer:** Use CloudTrail and simulator, check boundary/session/SCP/resource/endpoint/KMS policy, correct the smallest missing permission.
- **Follow-up questions:** No CloudTrail event? **Troubleshooting / architecture angle:** Confirm endpoint/account/region. **Memory hook:** PARC—principal, action, resource, context.

### Q038 — Explain `iam:PassRole`.
- **What interviewer is testing:** Privilege delegation. **Short answer:** It lets a principal configure a service to use a role; it does not assume that role.
- **Strong answer:** Restrict role ARN/path and `iam:PassedToService`; otherwise users may pass a powerful role to exploitable compute.
- **Follow-up questions:** CloudTrail event? **Troubleshooting / architecture angle:** Inspect create/update API. **Memory hook:** Pass to service.

### Q039 — How does cross-account access work?
- **What interviewer is testing:** Dual-side policy. **Short answer:** A trusted principal assumes a role or accesses a resource whose policy permits it, with both accounts’ controls satisfied.
- **Strong answer:** Prefer role sessions, narrow principal/org/external ID/source conditions, resource permissions, logging, and explicit ownership.
- **Follow-up questions:** S3 resource policy? **Troubleshooting / architecture angle:** Draw caller and resource account. **Memory hook:** Both sides agree.

### Q040 — Why use external ID?
- **What interviewer is testing:** Confused deputy. **Short answer:** It helps a third-party role distinguish the intended customer relationship and resist confused-deputy attacks.
- **Strong answer:** The third party should generate/manage unique values; still constrain principal, permissions, sessions, and audit.
- **Follow-up questions:** Is it secret? **Troubleshooting / architecture angle:** Test trust conditions. **Memory hook:** Customer binding, not password.

### Q041 — What is role chaining?
- **What interviewer is testing:** STS session nuance. **Short answer:** An assumed role uses its session to assume another role, with session-duration and policy implications.
- **Strong answer:** Minimize chains, understand source identity/session tags, max duration, trust, and auditability; prefer direct federation where possible.
- **Follow-up questions:** Duration limit? **Troubleshooting / architecture angle:** Inspect CloudTrail session issuer. **Memory hook:** Session assumes session.

### Q042 — Workload identity on EC2?
- **What interviewer is testing:** Credential hygiene. **Short answer:** Attach an instance profile role and retrieve temporary credentials through IMDS; avoid static keys.
- **Strong answer:** Require IMDSv2, limit hop/access, scope role, monitor sessions, and separate applications when host sharing weakens isolation.
- **Follow-up questions:** SSRF risk? **Troubleshooting / architecture angle:** Verify credential provider chain. **Memory hook:** Profile, not key file.

### Q043 — EKS Pod Identity versus IRSA?
- **What interviewer is testing:** Current EKS identity. **Short answer:** Both map service accounts to IAM roles; Pod Identity uses EKS associations/Auth agent, IRSA uses OIDC web identity.
- **Strong answer:** Choose based on SDK/add-on support, trust/cluster portability, operating model, and cross-account pattern; avoid broad node roles.
- **Follow-up questions:** Blue/green impact? **Troubleshooting / architecture angle:** Inspect service account/session. **Memory hook:** Agent association versus OIDC token.

### Q044 — ECS task role versus execution role?
- **What interviewer is testing:** Runtime boundary. **Short answer:** Task role is for application AWS calls; execution role is for ECS agent actions such as image pull/logs/secrets.
- **Strong answer:** Scope each task/service separately, check stopped reason and CloudTrail using the correct principal.
- **Follow-up questions:** EC2 container role? **Troubleshooting / architecture angle:** Who made the call? **Memory hook:** App task; agent execution.

### Q045 — How should CI access AWS?
- **What interviewer is testing:** Supply-chain identity. **Short answer:** Federate via OIDC to short-lived, environment-specific roles; do not store access keys.
- **Strong answer:** Constrain repository/branch/environment claims, session duration, permissions, protected approvals, and CloudTrail attribution.
- **Follow-up questions:** Fork PR? **Troubleshooting / architecture angle:** Treat source context as untrusted. **Memory hook:** Exchange identity, not secret.

### Q046 — Resource policy versus identity policy?
- **What interviewer is testing:** Policy placement. **Short answer:** Identity policy attaches to a principal; resource policy attaches to a supported resource and names principals.
- **Strong answer:** Effective access depends on same/cross-account semantics plus boundaries, SCPs, conditions, and explicit denies.
- **Follow-up questions:** Which services? **Troubleshooting / architecture angle:** Inspect both sides. **Memory hook:** Caller versus object.

### Q047 — What are session tags?
- **What interviewer is testing:** ABAC/federation. **Short answer:** Attributes passed into a role session can drive policy conditions and attribution.
- **Strong answer:** Control who may set/transit tags, source values from trusted identity claims, and avoid user-controlled privilege escalation.
- **Follow-up questions:** Transitive tags? **Troubleshooting / architecture angle:** Validate trust and tag keys. **Memory hook:** Trusted attributes on session.

### Q048 — How do you design break-glass access?
- **What interviewer is testing:** Resilience and governance. **Short answer:** A tightly controlled, monitored emergency role with strong authentication, short duration, approval/alerting, and post-use review.
- **Strong answer:** Keep it independent of likely failure paths, test access, restrict actions, log immutably, rotate controls, and never use routinely.
- **Follow-up questions:** If federation is down? **Troubleshooting / architecture angle:** Exercise the procedure. **Memory hook:** Available, narrow, noisy.

### Q049 — IAM authorization with KMS?
- **What interviewer is testing:** Multi-policy interaction. **Short answer:** IAM permission may not suffice; KMS key policy/grants and encryption context also govern use.
- **Strong answer:** Check correct key/region/state, service integration, identity/key policies, grants, SCP/endpoint policy, and context.
- **Follow-up questions:** Alias versus key ARN? **Troubleshooting / architecture angle:** Trace encrypt/decrypt call. **Memory hook:** Identity plus key authority.

### Q050 — How do you detect unused access?
- **What interviewer is testing:** Continuous least privilege. **Short answer:** Combine access last-used data, CloudTrail, Access Analyzer, reviews, and controlled removal tests.
- **Strong answer:** Observe a representative period including rare recovery jobs; stage removal with owner, rollback, and exception expiry.
- **Follow-up questions:** Data-event cost? **Troubleshooting / architecture angle:** Evidence window matters. **Memory hook:** Observe before subtracting.

## S3 — 15

### Q051 — S3 versus EBS versus EFS?
- **What interviewer is testing:** Storage semantics. **Short answer:** S3 is object/API, EBS is AZ-scoped block, EFS is regional shared NFS.
- **Strong answer:** Choose by access protocol/consistency first, then latency, throughput, sharing, durability/availability, lifecycle, backup, and cost.
- **Follow-up questions:** Database on EFS? **Troubleshooting / architecture angle:** Workload IO pattern. **Memory hook:** Object, block, file.

### Q052 — Explain S3 consistency.
- **What interviewer is testing:** Current behavior. **Short answer:** S3 provides strong read-after-write consistency for object writes/deletes and listings.
- **Strong answer:** Strong storage consistency does not coordinate concurrent writers; use conditional requests/version-aware workflows for races.
- **Follow-up questions:** Cross-Region replication? **Troubleshooting / architecture angle:** Separate local API consistency from async replication. **Memory hook:** Strong read, not distributed lock.

### Q053 — How do you secure an S3 bucket?
- **What interviewer is testing:** Defense in depth. **Short answer:** Block public access, least-privilege policies, controlled ownership, TLS/encryption, versioning/recovery, audit, and lifecycle.
- **Strong answer:** Include organization/endpoint conditions where suitable, KMS policy, data events by risk, Access Analyzer, retention and tested restore.
- **Follow-up questions:** Public exception? **Troubleshooting / architecture angle:** Enumerate every access path. **Memory hook:** Access, protect, observe, recover.

### Q054 — SSE-S3 versus SSE-KMS?
- **What interviewer is testing:** Encryption tradeoff. **Short answer:** Both encrypt at rest; SSE-KMS adds customer-controlled key policy/audit and KMS request considerations.
- **Strong answer:** Choose from separation, cross-account access, compliance, rotation/control, cost/quota, and operational recovery—not “more secure” alone.
- **Follow-up questions:** Bucket keys? **Troubleshooting / architecture angle:** Include KMS authorization. **Memory hook:** Encryption plus key control.

### Q055 — What does versioning protect?
- **What interviewer is testing:** Recovery semantics. **Short answer:** It retains versions so overwrites/deletes can be recovered; it increases retention cost and needs lifecycle.
- **Strong answer:** Define noncurrent expiration, delete-marker handling, replication, privileged permanent deletion, and restoration workflow.
- **Follow-up questions:** MFA Delete? **Troubleshooting / architecture angle:** Restore a sampled prefix. **Memory hook:** Delete becomes a version.

### Q056 — S3 replication design?
- **What interviewer is testing:** DR/data movement. **Short answer:** Configure source/destination versioning, rule scope, destination/KMS permissions, metrics, and recovery/failback procedure.
- **Strong answer:** Replication is asynchronous; define RPO, ownership, delete/metadata behavior, existing-object handling, and independent protection from logical corruption.
- **Follow-up questions:** RTC? **Troubleshooting / architecture angle:** Measure replication lag. **Memory hook:** Copy is async policy.

### Q057 — What is S3 Object Lock?
- **What interviewer is testing:** Immutability. **Short answer:** WORM retention/legal hold on object versions, with governance and compliance modes.
- **Strong answer:** Tie retention to legal requirements, isolate administration, understand override/delete consequences, KMS lifecycle, replication, and recovery testing.
- **Follow-up questions:** Versioning? **Troubleshooting / architecture angle:** Test authorized and denied deletion. **Memory hook:** Retention on versions.

### Q058 — How do lifecycle policies work?
- **What interviewer is testing:** Cost/retention. **Short answer:** Rules transition or expire current/noncurrent objects and multipart uploads based on filters/time.
- **Strong answer:** Model minimum duration/retrieval charges, rule precedence, delete markers, replication, legal holds, and restoration lead time.
- **Follow-up questions:** Immediate transition? **Troubleshooting / architecture angle:** Cost model with access distribution. **Memory hook:** Age drives class/action.

### Q059 — How do presigned URLs work?
- **What interviewer is testing:** Delegated access. **Short answer:** A signer grants time-limited permission for a specific S3 request using its own authority.
- **Strong answer:** Keep expiry short, exact method/key/content constraints, protect the URL as a bearer capability, and consider revocation via credential/policy changes.
- **Follow-up questions:** Upload limits? **Troubleshooting / architecture angle:** Threat-model leakage. **Memory hook:** Temporary signed request.

### Q060 — Why does S3 return AccessDenied?
- **What interviewer is testing:** Policy troubleshooting. **Short answer:** Identify principal/action/resource then inspect identity, bucket/access-point, endpoint, organization, ownership and KMS controls.
- **Strong answer:** Use request ID/CloudTrail, distinguish bucket versus object ARN and explicit deny, correct only the missing tuple.
- **Follow-up questions:** 403 versus 404? **Troubleshooting / architecture angle:** Avoid broad temporary allow. **Memory hook:** PARC plus KMS.

### Q061 — How do you optimize large uploads?
- **What interviewer is testing:** Transfer design. **Short answer:** Use multipart upload with controlled parallelism, retries/checksums, and cleanup of incomplete uploads.
- **Strong answer:** Benchmark object size, client CPU/network, endpoint/NAT, concurrency, KMS and request cost; resume failed parts idempotently.
- **Follow-up questions:** Part size? **Troubleshooting / architecture angle:** Measure bottleneck. **Memory hook:** Split, parallel, verify.

### Q062 — S3 event processing semantics?
- **What interviewer is testing:** Event-driven reliability. **Short answer:** Design consumers for duplicate and out-of-order notifications using idempotency and durable state.
- **Strong answer:** Filter events, decouple through queue where appropriate, use DLQ/redrive, record object version/sequencer, and reconcile missed work.
- **Follow-up questions:** Exactly once? **Troubleshooting / architecture angle:** Failure/replay test. **Memory hook:** Event is a hint; process idempotently.

### Q063 — How do you recover mass deletion?
- **What interviewer is testing:** Incident recovery. **Short answer:** Stop writers, preserve evidence, identify scope/versions, restore to controlled location, validate, then resume.
- **Strong answer:** Use versioning/Object Lock/replica/backup according to runbook, determine compromised identity, and fix destructive permissions before cutover.
- **Follow-up questions:** Delete markers? **Troubleshooting / architecture angle:** Automate manifest-based restore. **Memory hook:** Contain, identify, restore, validate.

### Q064 — Access point versus bucket policy?
- **What interviewer is testing:** Access at scale. **Short answer:** Access points provide named endpoints and policies for specific applications while bucket policy remains resource-wide control.
- **Strong answer:** Use access points to separate use cases/network origins, but evaluate combined policy, Block Public Access, ownership, and KMS.
- **Follow-up questions:** VPC-only? **Troubleshooting / architecture angle:** One policy contract per application. **Memory hook:** Per-use doorway.

### Q065 — How do you control S3 cost?
- **What interviewer is testing:** Operational economics. **Short answer:** Attribute storage, requests, retrieval, replication and transfer; apply lifecycle, compression/format, caching and cleanup.
- **Strong answer:** Use Storage Lens/inventory and access evidence; avoid transitions whose minimum durations/retrieval costs exceed benefit.
- **Follow-up questions:** Small objects? **Troubleshooting / architecture angle:** Unit economics. **Memory hook:** Bytes, requests, movement, retention.

## EC2, Auto Scaling, and load balancing — 15

### Q066 — How do you choose an EC2 instance type?
- **What interviewer is testing:** Workload fit. **Short answer:** Measure CPU, memory, network/EBS bandwidth, architecture, local storage, latency and price under representative load.
- **Strong answer:** Right-size from percentiles and bottlenecks, test multiple families, account for burst behavior, quotas/capacity and licensing.
- **Follow-up questions:** Graviton? **Troubleshooting / architecture angle:** Benchmark, do not guess. **Memory hook:** Fit the bottleneck.

### Q067 — On-Demand, Savings Plans, and Spot?
- **What interviewer is testing:** Capacity economics. **Short answer:** On-Demand is flexible; commitments discount stable use; Spot discounts interruptible spare capacity.
- **Strong answer:** Establish baseline, commit conservatively, diversify Spot and handle interruption; cost choices must preserve required capacity and recovery.
- **Follow-up questions:** Reserved Instances? **Troubleshooting / architecture angle:** Utilization and interruption model. **Memory hook:** Flexible, committed, interruptible.

### Q068 — What makes an AMI production-ready?
- **What interviewer is testing:** Immutable operations. **Short answer:** Minimal, patched, scanned, versioned, reproducibly built, boot-tested and traceable with no secrets.
- **Strong answer:** Promote through environments, validate architecture/drivers/agent, deprecate old images, and roll via ASG with rollback.
- **Follow-up questions:** Patch cadence? **Troubleshooting / architecture angle:** Launch canary. **Memory hook:** Bake, test, replace.

### Q069 — Instance store versus EBS?
- **What interviewer is testing:** Storage durability. **Short answer:** Instance store is host-local ephemeral high-performance storage; EBS is persistent network block storage in an AZ.
- **Strong answer:** Use instance store for reconstructible/cache/scratch data with replication; EBS for persistent volumes with snapshots and performance planning.
- **Follow-up questions:** Stop behavior? **Troubleshooting / architecture angle:** Design for host loss. **Memory hook:** Ephemeral local versus persistent block.

### Q070 — Explain EBS performance dimensions.
- **What interviewer is testing:** IO fundamentals. **Short answer:** IOPS, throughput, latency, IO size, queue depth, volume and instance ceilings interact.
- **Strong answer:** Throughput roughly equals IOPS times IO size; correlate CloudWatch with `iostat`, filesystem, application waits and instance bandwidth.
- **Follow-up questions:** High IOPS, slow app? **Troubleshooting / architecture angle:** Find limiting minimum. **Memory hook:** IO size connects IOPS to bytes.

### Q071 — System versus instance status check?
- **What interviewer is testing:** Failure isolation. **Short answer:** System check indicates underlying infrastructure path; instance check points to guest OS/network configuration.
- **Strong answer:** Use console/system logs, SSM/serial access, attached-volume health, recovery/replacement based on immutable design.
- **Follow-up questions:** Both fail? **Troubleshooting / architecture angle:** Preserve evidence, restore service. **Memory hook:** Host side versus guest side.

### Q072 — How do you secure instance metadata?
- **What interviewer is testing:** Credential protection. **Short answer:** Require IMDSv2, limit hop/access, patch SSRF, use narrow roles, and prevent containers from reaching metadata unnecessarily.
- **Strong answer:** Metadata hardening reduces but does not replace application isolation and least privilege; monitor credential/session use.
- **Follow-up questions:** Hop limit? **Troubleshooting / architecture angle:** Test from workload. **Memory hook:** Protect the credential endpoint.

### Q073 — How does target tracking scaling work?
- **What interviewer is testing:** Feedback control. **Short answer:** It adjusts capacity to keep a selected metric near a target, considering warm-up and scale-in behavior.
- **Strong answer:** Choose a demand-per-capacity metric, define min/max, startup delay, downstream limits, and test step response/oscillation.
- **Follow-up questions:** CPU unsuitable when? **Troubleshooting / architecture angle:** Graph demand versus capacity. **Memory hook:** Metric must divide with scale.

### Q074 — Why might ASG fail to launch?
- **What interviewer is testing:** Capacity diagnosis. **Short answer:** Check scaling activity for quota, subnet IP, AZ capacity, launch template, AMI/KMS, IAM, or instance-type errors.
- **Strong answer:** Diversify types/AZs, keep known-good template versions, launch canaries, and alarm before maximum or IP exhaustion.
- **Follow-up questions:** Spot failure? **Troubleshooting / architecture angle:** Read exact activity reason. **Memory hook:** Desired is not available.

### Q075 — ALB versus NLB?
- **What interviewer is testing:** Load balancer choice. **Short answer:** ALB offers Layer-7 HTTP routing; NLB handles Layer-4 TCP/UDP/TLS performance and static-address needs.
- **Strong answer:** Select by protocol, routing, source IP, TLS, targets, health, latency, scale, private connectivity and cost.
- **Follow-up questions:** gRPC? PrivateLink? **Troubleshooting / architecture angle:** Start with protocol. **Memory hook:** L7 rules versus L4 flow.

### Q076 — What makes a good health check?
- **What interviewer is testing:** Availability design. **Short answer:** Fast, reliable readiness signal that proves a target can serve without creating a shared-dependency cascade.
- **Strong answer:** Separate startup/liveness/readiness/SLO; tune interval/threshold/grace and verify actual user paths synthetically.
- **Follow-up questions:** Database check? **Troubleshooting / architecture angle:** Failure-mode test. **Memory hook:** Admit safe traffic.

### Q077 — Diagnose ALB 502/503/504.
- **What interviewer is testing:** Layer mapping. **Short answer:** 502 suggests invalid/reset target response, 503 unavailable healthy targets, 504 target response timeout.
- **Strong answer:** Confirm origin in access logs, target health, app logs/traces, protocol/port, capacity and dependency/timeout chain.
- **Follow-up questions:** ALB-generated versus target code? **Troubleshooting / architecture angle:** Request ID and timestamps. **Memory hook:** Bad response, no target, slow target.

### Q078 — Rolling versus blue/green deployment?
- **What interviewer is testing:** Release tradeoffs. **Short answer:** Rolling replaces capacity incrementally; blue/green builds parallel environment and shifts traffic.
- **Strong answer:** Blue/green gives faster compute rollback but costs duplicate capacity and cannot automatically reverse data/schema side effects.
- **Follow-up questions:** Canary? **Troubleshooting / architecture angle:** Define rollback evidence. **Memory hook:** Replace in place versus switch environments.

### Q079 — Why high CPU after scaling out?
- **What interviewer is testing:** Bottleneck reasoning. **Short answer:** Load may be uneven, metric lagged, state/session pinned, dependency saturated, or workload not horizontally divisible.
- **Strong answer:** Compare per-target requests/latency, connection behavior, hot keys, queues, GC, database and cache; fix cause, not only capacity.
- **Follow-up questions:** Sticky sessions? **Troubleshooting / architecture angle:** Compare affected targets. **Memory hook:** More hosts do not divide every bottleneck.

### Q080 — How do you patch an EC2 fleet?
- **What interviewer is testing:** Safe operations. **Short answer:** Build/test a patched image and roll through ASG waves with health/SLO gates and rollback; use managed patching when model requires.
- **Strong answer:** Inventory urgency, canary, capacity surge, state drain, reboot, compliance evidence, exception handling, and emergency process.
- **Follow-up questions:** Zero-day? **Troubleshooting / architecture angle:** Risk-based rings. **Memory hook:** Patch, prove, replace.

## EKS — 25

### Q081 — What does EKS manage?
- **What interviewer is testing:** Responsibility boundary. **Short answer:** AWS manages the Kubernetes control plane; customer owns workloads, access, data plane/Fargate configuration, add-ons, security and upgrades.
- **Strong answer:** Name control-plane availability plus customer API compatibility, nodes, CNI/CSI/DNS, RBAC, policies, telemetry and recovery.
- **Follow-up questions:** etcd access? **Troubleshooting / architecture angle:** Shared responsibility. **Memory hook:** Managed control, owned platform.

### Q082 — Managed node group versus Fargate?
- **What interviewer is testing:** Compute tradeoff. **Short answer:** Node groups offer instance/daemon/storage control; Fargate removes nodes with per-pod constraints and economics.
- **Strong answer:** Compare workload compatibility, DaemonSets, privileged access, startup, isolation, bin-packing, Spot, observability and cost.
- **Follow-up questions:** System pods? **Troubleshooting / architecture angle:** Requirements before abstraction. **Memory hook:** Control versus serverless pods.

### Q083 — How does VPC CNI work?
- **What interviewer is testing:** Pod networking. **Short answer:** It assigns VPC-routable addresses to pods through ENIs/prefixes on nodes.
- **Strong answer:** Plan subnet IPs, ENI/pod density, prefix delegation, security groups/network policy, warm pools and scaling failure signals.
- **Follow-up questions:** IP exhaustion? **Troubleshooting / architecture angle:** Pod-to-ENI path. **Memory hook:** Pods consume network capacity.

### Q084 — Service versus Ingress?
- **What interviewer is testing:** Kubernetes networking. **Short answer:** Service provides stable internal discovery/load balancing; Ingress declares HTTP routing implemented by a controller.
- **Strong answer:** Trace DNS -> controller-created LB -> listener/rule -> Service -> ready endpoints -> pod targetPort.
- **Follow-up questions:** Gateway API? **Troubleshooting / architecture angle:** Declaration needs controller. **Memory hook:** Stable service, external routes.

### Q085 — Diagnose a Pending pod.
- **What interviewer is testing:** Scheduler method. **Short answer:** Read events, then check requests, node capacity, selectors/affinity, taints, topology, quotas, PVC and IPs.
- **Strong answer:** Verify autoscaler decision, cloud capacity and pod density; change the smallest invalid constraint or add compatible capacity.
- **Follow-up questions:** Preemption? **Troubleshooting / architecture angle:** Scheduler event is first evidence. **Memory hook:** Constraint meets capacity.

### Q086 — Diagnose PVC Pending.
- **What interviewer is testing:** Storage troubleshooting. **Short answer:** Inspect PVC events, StorageClass, access mode, CSI health/IAM, binding mode, topology, quota and KMS.
- **Strong answer:** `WaitForFirstConsumer` ties provisioning to pod scheduling/AZ; avoid deleting stateful objects blindly.
- **Follow-up questions:** EBS multi-attach? **Troubleshooting / architecture angle:** Claim -> class -> CSI -> cloud volume. **Memory hook:** Class, controller, zone.

### Q087 — CrashLoopBackOff workflow?
- **What interviewer is testing:** Pod diagnosis. **Short answer:** Inspect events, previous logs, exit code, command/config/secrets, OOM and probes.
- **Strong answer:** Compare last good version, reproduce configuration, check dependencies and resource limits, then rollback or correct narrowly.
- **Follow-up questions:** Backoff meaning? **Troubleshooting / architecture angle:** Previous container evidence. **Memory hook:** Exit before restart.

### Q088 — Readiness versus liveness versus startup probe?
- **What interviewer is testing:** Health semantics. **Short answer:** Readiness controls traffic, liveness restarts, startup delays the other probes during initialization.
- **Strong answer:** Use distinct inexpensive endpoints and thresholds; wrong liveness causes restart cascades, wrong readiness sends traffic too early.
- **Follow-up questions:** Dependency checks? **Troubleshooting / architecture angle:** Simulate failure. **Memory hook:** Admit, restart, initialize.

### Q089 — Requests versus limits?
- **What interviewer is testing:** Resource management. **Short answer:** Requests drive scheduling; limits constrain runtime—CPU throttles and memory excess can OOM.
- **Strong answer:** Size from observed percentiles/load tests, account for node/system headroom, QoS and autoscaling signals.
- **Follow-up questions:** No limit? **Troubleshooting / architecture angle:** Correlate throttling/OOM. **Memory hook:** Reserve versus cap.

### Q090 — What is a PodDisruptionBudget?
- **What interviewer is testing:** Availability nuance. **Short answer:** It limits voluntary concurrent pod disruptions; it does not guarantee replicas or protect involuntary failures.
- **Strong answer:** Combine with topology, capacity, replicas and graceful shutdown; impossible PDBs can block node drains/upgrades.
- **Follow-up questions:** `minAvailable`? **Troubleshooting / architecture angle:** Test a drain. **Memory hook:** Voluntary disruption budget.

### Q091 — Design zero-downtime EKS upgrade.
- **What interviewer is testing:** Senior operations. **Short answer:** Preflight deprecated APIs/add-ons, upgrade control plane then add-ons/nodes in tested waves with surge, PDBs and SLO gates.
- **Strong answer:** Rehearse nonprod, canary workloads, cordon/drain, validate DNS/storage/ingress/autoscaling; blue/green for stronger rollback.
- **Follow-up questions:** Roll back control plane? **Troubleshooting / architecture angle:** Compatibility matrix. **Memory hook:** API, add-ons, nodes, workloads.

### Q092 — How do you upgrade worker nodes?
- **What interviewer is testing:** Disruption control. **Short answer:** Add updated capacity, cordon/drain old nodes, reschedule safely, validate, then terminate in waves.
- **Strong answer:** Respect PDBs, local/stateful data, DaemonSets, topology, IPs, maxUnavailable, bootstrap and capacity availability.
- **Follow-up questions:** Stuck drain? **Troubleshooting / architecture angle:** One node canary. **Memory hook:** Add, drain, verify, remove.

### Q093 — Cluster Autoscaler versus Karpenter?
- **What interviewer is testing:** Capacity provisioning concepts. **Short answer:** Both add nodes for unschedulable pods; Cluster Autoscaler scales predefined groups, Karpenter directly selects/provisions suitable capacity.
- **Strong answer:** Compare flexibility, consolidation, constraints, disruption, Spot handling, governance and operational maturity for current versions.
- **Follow-up questions:** HPA interaction? **Troubleshooting / architecture angle:** Pod demand to cloud capacity. **Memory hook:** Groups versus direct provisioning.

### Q094 — HPA versus VPA?
- **What interviewer is testing:** Scaling dimensions. **Short answer:** HPA changes replica count; VPA recommends/changes pod requests; combined use requires avoiding signal conflicts.
- **Strong answer:** Select meaningful metrics, stabilization behavior, startup and downstream constraints; load-test scaling loops.
- **Follow-up questions:** CPU utilization denominator? **Troubleshooting / architecture angle:** Demand, pod size, node capacity. **Memory hook:** More pods versus bigger pods.

### Q095 — Secure EKS workload identity.
- **What interviewer is testing:** Least privilege. **Short answer:** Assign role per service account using Pod Identity or IRSA; do not rely on broad node credentials.
- **Strong answer:** Constrain association/trust, namespace/service account, role policy and session; protect metadata and audit actual assumed sessions.
- **Follow-up questions:** SDK support? **Troubleshooting / architecture angle:** Verify credential chain in pod. **Memory hook:** Pod gets its own role.

### Q096 — EKS authentication versus authorization?
- **What interviewer is testing:** Access model. **Short answer:** AWS identity authenticates to cluster; Kubernetes RBAC/access configuration authorizes API actions.
- **Strong answer:** Use access entries/current supported mechanism, federated roles, least RBAC, separated admins, audit logs and break-glass.
- **Follow-up questions:** `kubectl auth can-i`? **Troubleshooting / architecture angle:** Test exact impersonated action. **Memory hook:** Enter, then authorize.

### Q097 — How do you enforce network isolation?
- **What interviewer is testing:** Layered network policy. **Short answer:** Use namespaces/account/cluster boundaries, SGs and Kubernetes network policies with a supporting data plane.
- **Strong answer:** Default-deny ingress/egress, explicitly allow DNS/dependencies, validate enforcement, observe flows, and separate high-trust workloads strongly.
- **Follow-up questions:** SG for pods? **Troubleshooting / architecture angle:** Test real packet path. **Memory hook:** Policy needs enforcement.

### Q098 — Why is a Service not routing traffic?
- **What interviewer is testing:** Endpoint mapping. **Short answer:** Check selector and ready endpoints, ports/targetPort, pod listener, DNS, kube networking and policy.
- **Strong answer:** Test pod IP then ClusterIP then ingress path; compare EndpointSlices and readiness events.
- **Follow-up questions:** Headless Service? **Troubleshooting / architecture angle:** Narrow hop by hop. **Memory hook:** No endpoints, no service.

### Q099 — How do you troubleshoot CoreDNS?
- **What interviewer is testing:** Cluster DNS. **Short answer:** Query from pod, inspect CoreDNS pods/logs/metrics/config/endpoints, resources, network policy and upstream Resolver.
- **Strong answer:** Compare cached/uncached and TCP/UDP, affected nodes/AZs, throttling, forwarding loops and node DNS configuration.
- **Follow-up questions:** `ndots`? **Troubleshooting / architecture angle:** Query volume and suffix expansion. **Memory hook:** Client, CoreDNS, upstream.

### Q100 — How do you handle secrets in EKS?
- **What interviewer is testing:** Secret lifecycle. **Short answer:** Use workload identity to retrieve managed secrets or a controlled CSI/operator pattern; encrypt and restrict Kubernetes secrets.
- **Strong answer:** Define rotation/cache, RBAC/etcd encryption, namespace isolation, no env/log leaks, audit and failure behavior.
- **Follow-up questions:** Secret refresh? **Troubleshooting / architecture angle:** Rotation end-to-end. **Memory hook:** Retrieve, restrict, rotate.

### Q101 — What causes OOMKilled?
- **What interviewer is testing:** Runtime memory. **Short answer:** Container exceeded its memory cgroup limit or node pressure led to termination/eviction; inspect status and metrics.
- **Strong answer:** Compare working set, leak/heap, request/limit, concurrency and node pressure; stabilize then profile and right-size.
- **Follow-up questions:** Exit 137? **Troubleshooting / architecture angle:** Container versus node. **Memory hook:** Limit, leak, load.

### Q102 — How do you design EKS observability?
- **What interviewer is testing:** Platform operations. **Short answer:** Collect control-plane audit, Kubernetes events, node/container logs, workload RED, resource USE and traces with correlation.
- **Strong answer:** Standard labels but cap cardinality, deployment events, SLO alerts, retention/access, cost, and runbooks.
- **Follow-up questions:** Missing pod logs? **Troubleshooting / architecture angle:** Follow request/workload/node/AZ. **Memory hook:** State, signal, change.

### Q103 — How do you secure the software supply chain?
- **What interviewer is testing:** End-to-end security. **Short answer:** Protect source/build, pin dependencies, scan/SBOM, sign provenance, allow trusted registries, verify at admission and deploy by digest.
- **Strong answer:** Isolate untrusted builds, short-lived roles, exception expiry, continuous runtime inventory, and tested revocation/rebuild.
- **Follow-up questions:** Scan false positives? **Troubleshooting / architecture angle:** Risk-based policy. **Memory hook:** Source to running digest.

### Q104 — How do you run stateful workloads on EKS?
- **What interviewer is testing:** Storage/availability. **Short answer:** Use suitable CSI storage, StatefulSets, topology-aware provisioning, backups and application-native replication/recovery.
- **Strong answer:** Define access mode, AZ coupling, rescheduling, snapshots versus consistent backup, disruption, upgrade and restore testing.
- **Follow-up questions:** EBS versus EFS? **Troubleshooting / architecture angle:** Failure of node/AZ/cluster. **Memory hook:** Stateful means data lifecycle.

### Q105 — EKS versus ECS?
- **What interviewer is testing:** Platform selection. **Short answer:** ECS is simpler AWS-native orchestration; EKS provides Kubernetes APIs/ecosystem and greater platform flexibility.
- **Strong answer:** Compare required portability/ecosystem, team skill, security model, operations, workload constraints and total cost—not feature count.
- **Follow-up questions:** Fargate on both? **Troubleshooting / architecture angle:** Organizational fit. **Memory hook:** Simplicity versus Kubernetes standard.

## ECS — 10

### Q106 — Task, task definition, service, cluster?
- **What interviewer is testing:** ECS model. **Short answer:** Definition is versioned spec, task is running copy, service maintains tasks/deployments, cluster is capacity/logical grouping.
- **Strong answer:** Add capacity provider, networking, role, LB and autoscaling relationships.
- **Follow-up questions:** Standalone task? **Troubleshooting / architecture angle:** Trace desired to running. **Memory hook:** Spec, copy, keeper, pool.

### Q107 — Fargate versus ECS on EC2?
- **What interviewer is testing:** Compute model. **Short answer:** Fargate removes host operations; EC2 gives instance control, broader patterns and potential steady-scale efficiency.
- **Strong answer:** Compare isolation, privileged/agent needs, scaling, startup, purchase models, utilization, patching and cost.
- **Follow-up questions:** Spot? **Troubleshooting / architecture angle:** Total operational cost. **Memory hook:** Serverless tasks versus owned fleet.

### Q108 — Why does a task stop before starting?
- **What interviewer is testing:** Launch troubleshooting. **Short answer:** Inspect stopped reason for image/platform, execution role, network/DNS, secret/KMS, log driver, capacity or resource error.
- **Strong answer:** Validate exact task revision and subnet/endpoint path; correct the responsible layer and redeploy immutable revision.
- **Follow-up questions:** CannotPullContainerError? **Troubleshooting / architecture angle:** Agent prerequisites. **Memory hook:** Reason before retry.

### Q109 — How do ECS deployments roll back?
- **What interviewer is testing:** Release safety. **Short answer:** Use deployment circuit breaker/alarms or blue-green controls to stop and restore a known-good task definition.
- **Strong answer:** Configure health grace, min/max healthy, spare capacity, deregistration, immutable image and database compatibility.
- **Follow-up questions:** Bake time? **Troubleshooting / architecture angle:** Define failure threshold. **Memory hook:** Detect, stop, restore revision.

### Q110 — ECS service scaling design?
- **What interviewer is testing:** Capacity loop. **Short answer:** Scale task count on demand-per-task signals and ensure capacity provider/network/downstream can satisfy it.
- **Strong answer:** Use request or queue backlog metrics, warm-up/cooldown, min/max, scheduled baseline and downstream protection.
- **Follow-up questions:** Capacity provider strategy? **Troubleshooting / architecture angle:** Service and host capacity separately. **Memory hook:** Tasks plus capacity.

### Q111 — ECS networking modes?
- **What interviewer is testing:** Connectivity. **Short answer:** `awsvpc` gives tasks ENIs and is required for Fargate; other EC2 modes share/map host networking differently.
- **Strong answer:** `awsvpc` improves identity/isolation but consumes IP/ENI capacity; validate target type, SG and port mapping.
- **Follow-up questions:** IP target? **Troubleshooting / architecture angle:** Trace task ENI. **Memory hook:** Task gets network identity.

### Q112 — Why is ECS target unhealthy?
- **What interviewer is testing:** LB integration. **Short answer:** Check container port/listener, target type, SG, health path/status, bind address, startup and application logs.
- **Strong answer:** Compare container health, ECS task health and target-group health; they are distinct signals.
- **Follow-up questions:** Grace period? **Troubleshooting / architecture angle:** Directly test target. **Memory hook:** Running is not ready.

### Q113 — How do tasks retrieve secrets?
- **What interviewer is testing:** Identity separation. **Short answer:** ECS can inject referenced Secrets Manager/Parameter Store values using execution-role permissions, or app retrieves at runtime using task role.
- **Strong answer:** Choose based on rotation/refresh, exposure, SDK behavior, least privilege, KMS and deployment restart semantics.
- **Follow-up questions:** Rotation without restart? **Troubleshooting / architecture angle:** Follow credential lifecycle. **Memory hook:** Agent injects; app retrieves.

### Q114 — What are capacity providers?
- **What interviewer is testing:** ECS capacity strategy. **Short answer:** They connect task placement to Fargate or Auto Scaling capacity with base and weighted strategies.
- **Strong answer:** Mix stable and Spot capacity according to interruption tolerance, scaling lag, AZ/type diversity and service criticality.
- **Follow-up questions:** Managed scaling? **Troubleshooting / architecture angle:** Desired tasks versus instances. **Memory hook:** Placement chooses capacity source.

### Q115 — How do you observe ECS?
- **What interviewer is testing:** Operability. **Short answer:** Correlate service events, task stopped reasons, container logs, LB health/access, utilization, deployment and traces.
- **Strong answer:** Add Container Insights where justified, request IDs, version/task dimensions, SLO alerts and logs for execution/runtime identity failures.
- **Follow-up questions:** High cardinality? **Troubleshooting / architecture angle:** Follow user request to task/dependency. **Memory hook:** Event, task, target, app.

## RDS and Aurora — 15

### Q116 — Multi-AZ versus read replica?
- **What interviewer is testing:** HA versus scale. **Short answer:** Multi-AZ is primarily availability/failover; read replicas asynchronously scale reads and may support DR patterns.
- **Strong answer:** Engine/deployment type matters; define replication, endpoint, consistency, promotion, lag and recovery behavior precisely.
- **Follow-up questions:** Can standby serve reads? **Troubleshooting / architecture angle:** Name requirement first. **Memory hook:** HA versus read scale.

### Q117 — RDS versus Aurora?
- **What interviewer is testing:** Database selection. **Short answer:** RDS manages standard engines; Aurora is AWS-designed MySQL/PostgreSQL-compatible architecture with distributed storage and cluster features.
- **Strong answer:** Compare compatibility, performance evidence, availability/failover, replicas/global needs, features, migration, skills and total cost.
- **Follow-up questions:** Serverless? **Troubleshooting / architecture angle:** Benchmark workload. **Memory hook:** Compatibility plus architecture.

### Q118 — How do automated backups and PITR work conceptually?
- **What interviewer is testing:** Recovery. **Short answer:** RDS retains backups and transaction logs within a window to restore a new database near a chosen time.
- **Strong answer:** Define retention, KMS, Region/account copies, restore duration, parameter/network configuration, validation and cutover.
- **Follow-up questions:** Does restore overwrite? **Troubleshooting / architecture angle:** Time the full restore. **Memory hook:** Restore creates new.

### Q119 — How do you troubleshoot a slow database?
- **What interviewer is testing:** Evidence method. **Short answer:** Start with transaction SLO, waits/top SQL/locks/connections, then CPU/memory/cache/storage and changes.
- **Strong answer:** Inspect execution plan, rows/index/statistics, temp spill, pool and dependency; stabilize safely and test correction on representative data.
- **Follow-up questions:** High CPU? **Troubleshooting / architecture angle:** Waits before resizing. **Memory hook:** Query, wait, resource, change.

### Q120 — What causes replica lag?
- **What interviewer is testing:** Replication mechanics. **Short answer:** Heavy writes, long transactions/DDL, insufficient replica compute/IO, blocked apply, network or heavy replica reads.
- **Strong answer:** Measure engine-specific lag and apply state, protect consistency-sensitive reads, scale/remove blocker, and set lag routing thresholds.
- **Follow-up questions:** Promotion risk? **Troubleshooting / architecture angle:** Compare generation versus apply rate. **Memory hook:** Replica cannot apply fast enough.

### Q121 — How do you design connection pooling?
- **What interviewer is testing:** Database protection. **Short answer:** Bound connections per app instance based on DB capacity, reuse them, time out leaks and prevent retry storms.
- **Strong answer:** Model total fleet maximum, transaction duration, server resources, failover refresh, RDS Proxy suitability and observability.
- **Follow-up questions:** Scale-out danger? **Troubleshooting / architecture angle:** Pool budget is global. **Memory hook:** App replicas multiply connections.

### Q122 — What is RDS Proxy for?
- **What interviewer is testing:** Managed connection proxy. **Short answer:** It pools/manages database connections to absorb churn and improve application failover behavior for supported use cases.
- **Strong answer:** It does not fix slow SQL or unlimited transactions; evaluate pinning, auth, latency, cost, engine support and pool configuration.
- **Follow-up questions:** Session state? **Troubleshooting / architecture angle:** Measure before/after connections. **Memory hook:** Smooth connections, not queries.

### Q123 — How do you perform a safe schema migration?
- **What interviewer is testing:** Deployment/data compatibility. **Short answer:** Use expand/contract: add compatible schema, deploy dual-compatible code, migrate/validate, switch, then remove later.
- **Strong answer:** Assess locks/table rewrite, throttle/backfill, observability, cancellation/rollback and replication impact on production-like data.
- **Follow-up questions:** Destructive rollback? **Troubleshooting / architecture angle:** Separate code and data reversibility. **Memory hook:** Expand, migrate, contract.

### Q124 — How do you encrypt and rotate DB credentials?
- **What interviewer is testing:** Secret lifecycle. **Short answer:** Store in Secrets Manager, authorize workload identity, rotate with overlapping stages, refresh pools and audit access.
- **Strong answer:** Separate admin/app users, TLS, KMS policy, failure rollback, client caching and end-to-end rotation test.
- **Follow-up questions:** IAM DB auth? **Troubleshooting / architecture angle:** Credential works after rotation. **Memory hook:** Rotate value and consumers.

### Q125 — RDS storage full: response?
- **What interviewer is testing:** Safe incident response. **Short answer:** Stop growth/load safely, identify logs/temp/table/index source, increase storage if appropriate, then correct retention/query/capacity.
- **Strong answer:** Watch autoscaling limits, free storage, transaction logs and engine behavior; avoid risky cleanup without backup and ownership.
- **Follow-up questions:** Can shrink? **Troubleshooting / architecture angle:** Growth rate and time to exhaustion. **Memory hook:** Stabilize space, find growth.

### Q126 — How do you test failover?
- **What interviewer is testing:** Resilience proof. **Short answer:** In a controlled window trigger supported failover and measure detection, DNS/pool reconnect, transaction behavior and user RTO.
- **Strong answer:** Define expected errors, bounded retries/idempotency, monitoring, stakeholders, abort/failback, and corrective actions.
- **Follow-up questions:** Data loss? **Troubleshooting / architecture angle:** Test application, not DB event only. **Memory hook:** Failover ends when users recover.

### Q127 — Snapshot copy across accounts/Regions?
- **What interviewer is testing:** DR/security. **Short answer:** Share/copy snapshots with compatible KMS permissions and destination-owned encryption, then configure network/parameters on restore.
- **Strong answer:** Automate retention and copy status, isolate recovery account, test engine/version/option dependencies and restore time.
- **Follow-up questions:** AWS-managed key? **Troubleshooting / architecture angle:** Key lifecycle is recovery dependency. **Memory hook:** Data copy plus key access.

### Q128 — How do you monitor RDS?
- **What interviewer is testing:** Database observability. **Short answer:** Track transaction SLO, connections, CPU/memory, storage/IO, replica lag, waits/top SQL, logs and events.
- **Strong answer:** Use engine-aware tooling/Performance Insights capabilities, Enhanced Monitoring where justified, deployment correlation and owned thresholds.
- **Follow-up questions:** Average latency? **Troubleshooting / architecture angle:** Tail and workload class. **Memory hook:** User, query, resource, replication.

### Q129 — When use Aurora Global Database?
- **What interviewer is testing:** Multi-Region data design. **Short answer:** For cross-Region read locality and low-RPO DR where Aurora compatibility, cost and promotion model fit.
- **Strong answer:** Define writer Region, replication lag, planned/unplanned failover, client endpoints, write forwarding/consistency, quotas and failback reconciliation.
- **Follow-up questions:** Active/active writes? **Troubleshooting / architecture angle:** Write ownership first. **Memory hook:** Global reads, controlled writer.

### Q130 — How do you reduce database cost safely?
- **What interviewer is testing:** Optimization without risk. **Short answer:** Right-size from utilization/waits, optimize queries/storage/connections, schedule nonprod, use commitments after stable baseline.
- **Strong answer:** Include replicas, backup retention, I/O, transfer and licenses; load/failover-test changes and preserve headroom/SLO.
- **Follow-up questions:** Downsize risk? **Troubleshooting / architecture angle:** Cost per transaction with resilience. **Memory hook:** Optimize work before capacity.

## CI/CD, Terraform, GitHub Actions, and Jenkins — 20

### Q131 — What makes a production CI/CD pipeline?
- **What interviewer is testing:** Delivery architecture. **Short answer:** Reviewed source builds one immutable artifact, proves controls, promotes with least privilege, verifies SLOs and can recover.
- **Strong answer:** Include provenance/SBOM, test/policy gates, environment approvals, concurrency, canary/blue-green, database compatibility and audit evidence.
- **Follow-up questions:** Speed versus control? **Troubleshooting / architecture angle:** Automate risk-proportional gates. **Memory hook:** Build once, promote, prove.

### Q132 — Why build once and promote?
- **What interviewer is testing:** Artifact integrity. **Short answer:** It ensures production receives the exact artifact tested, avoiding environment-specific rebuild drift.
- **Strong answer:** Identify by immutable digest, attach provenance/signature, separate config, restrict registry promotion, and record deployment mapping.
- **Follow-up questions:** Environment config? **Troubleshooting / architecture angle:** Verify running digest. **Memory hook:** Same bits, different config.

### Q133 — Canary versus blue/green?
- **What interviewer is testing:** Release tradeoff. **Short answer:** Canary sends a small traffic share to new version; blue/green prepares parallel environments and switches traffic.
- **Strong answer:** Canary needs representative traffic/metrics; blue/green needs duplicate capacity. Both require data compatibility and explicit promotion/rollback.
- **Follow-up questions:** Sticky sessions? **Troubleshooting / architecture angle:** Define success metrics. **Memory hook:** Sample traffic versus switch environment.

### Q134 — How do you secure pipeline credentials?
- **What interviewer is testing:** Supply-chain security. **Short answer:** Prefer OIDC/short-lived roles, least privilege per stage/environment, protected approvals and no secret output/artifacts.
- **Strong answer:** Isolate untrusted builds, rotate remaining secrets, pin actions/plugins, audit sessions and limit production network/data paths.
- **Follow-up questions:** Self-hosted runner? **Troubleshooting / architecture angle:** Threat-model PR code. **Memory hook:** Federate, scope, isolate.

### Q135 — What is Terraform state?
- **What interviewer is testing:** Core Terraform. **Short answer:** It maps configuration resource addresses to remote objects and metadata used for planning; it can contain sensitive values.
- **Strong answer:** Protect remote state with encryption, versioning, locking, least privilege, backups and one-writer controls; separate blast-radius states.
- **Follow-up questions:** Is it a cache? **Troubleshooting / architecture angle:** Losing state changes ownership mapping. **Memory hook:** Map, not mere cache.

### Q136 — How do you configure Terraform locking now?
- **What interviewer is testing:** Current backend knowledge. **Short answer:** For supported Terraform versions the S3 backend can use native lockfiles with `use_lockfile = true`; DynamoDB locking is deprecated.
- **Strong answer:** Also enable bucket versioning/encryption and CI concurrency; verify version-specific official docs and migrate locking deliberately.
- **Follow-up questions:** Force unlock? **Troubleshooting / architecture angle:** Prove no active writer. **Memory hook:** Lock plus versioned recovery.

### Q137 — Handle Terraform partial apply.
- **What interviewer is testing:** Recovery discipline. **Short answer:** Freeze writers, preserve logs/state, compare cloud reality and fresh plan, fix cause, review and reconcile.
- **Strong answer:** Successful changes are recorded; do not rerun blindly or edit state. Use import/state operations only with backup and exact ownership proof.
- **Follow-up questions:** Rollback? **Troubleshooting / architecture angle:** Desired convergence versus reversible change. **Memory hook:** Freeze, inspect, re-plan.

### Q138 — Recover corrupted Terraform state.
- **What interviewer is testing:** High-risk operations. **Short answer:** Stop applies, preserve versions, validate backend lineage/serial and real resources, restore known-good state or repair precisely.
- **Strong answer:** Use S3 versioning, refresh-only/normal plans cautiously, peer review, and fix locking/access root cause before apply.
- **Follow-up questions:** Manual edit? **Troubleshooting / architecture angle:** Never guess resource mapping. **Memory hook:** Preserve, compare, restore, prove.

### Q139 — How do you manage drift?
- **What interviewer is testing:** IaC governance. **Short answer:** Detect scheduled plans, determine whether change is authorized, then encode/import it or let reviewed code restore state.
- **Strong answer:** Restrict console mutation, record emergency changes, avoid automatic destructive reconciliation, and measure recurring drift causes.
- **Follow-up questions:** `ignore_changes`? **Troubleshooting / architecture angle:** Intent before convergence. **Memory hook:** Detect, decide, reconcile.

### Q140 — Refactor Terraform without recreation.
- **What interviewer is testing:** State/address safety. **Short answer:** Use `moved` blocks for address changes and import for existing objects; review plan until no unintended replacement.
- **Strong answer:** Pin versions, migrate in small steps, back up state, understand `for_each` keys and force-new attributes.
- **Follow-up questions:** Module rename? **Troubleshooting / architecture angle:** Address mapping proof. **Memory hook:** Move address, not resource.

### Q141 — How do you design Terraform modules?
- **What interviewer is testing:** Maintainability. **Short answer:** Cohesive opinionated modules with typed/validated inputs, useful outputs, versioning, documentation, examples and tests.
- **Strong answer:** Avoid mega-modules and exposing provider internals; define ownership, compatibility and upgrade path.
- **Follow-up questions:** Composition? **Troubleshooting / architecture angle:** Stable contract. **Memory hook:** Small interface, clear lifecycle.

### Q142 — Why is `sensitive = true` insufficient?
- **What interviewer is testing:** Secret handling. **Short answer:** It redacts selected CLI display but values can remain in state and plans.
- **Strong answer:** Pass references instead of secret values, retrieve at runtime, restrict state/artifacts and avoid secret-generating resources where possible.
- **Follow-up questions:** Environment variables? **Troubleshooting / architecture angle:** Trace every persistence location. **Memory hook:** Redacted is not absent.

### Q143 — GitHub Actions OIDC flow?
- **What interviewer is testing:** Federation. **Short answer:** Workflow obtains GitHub OIDC token; AWS trust validates claims; STS returns temporary role credentials.
- **Strong answer:** Grant `id-token: write`, constrain audience/subject/repository/environment, minimize role policy/session and audit CloudTrail.
- **Follow-up questions:** Branch condition? **Troubleshooting / architecture angle:** Inspect claims safely. **Memory hook:** Token exchange, temporary role.

### Q144 — `pull_request` versus `pull_request_target` risk?
- **What interviewer is testing:** Workflow security. **Short answer:** `pull_request_target` runs in privileged base context; executing untrusted PR code there can expose authority.
- **Strong answer:** Use unprivileged checks for fork code, never checkout/execute attacker content with secrets/write token, and gate trusted follow-on workflows.
- **Follow-up questions:** Artifact poisoning? **Troubleshooting / architecture angle:** Trust boundary per event. **Memory hook:** Target context is privileged.

### Q145 — Reusable workflow versus composite action?
- **What interviewer is testing:** GitHub Actions design. **Short answer:** Reusable workflows share jobs/governance; composite actions bundle steps within a caller job.
- **Strong answer:** Define explicit inputs/secrets/permissions, pin versions and use reusable workflows for central deployment policy.
- **Follow-up questions:** Nested permissions? **Troubleshooting / architecture angle:** Least privilege contract. **Memory hook:** Jobs versus steps.

### Q146 — How do you prevent concurrent deployments?
- **What interviewer is testing:** Serialization. **Short answer:** Use environment concurrency/locks so only one production mutation runs, with non-canceling semantics for critical applies.
- **Strong answer:** Coordinate pipeline deployment lock with Terraform state locking; handle stale runs, approvals, timeouts and idempotent resume.
- **Follow-up questions:** Multi-repository? **Troubleshooting / architecture angle:** One authoritative deployment queue. **Memory hook:** Two locks, one writer.

### Q147 — Jenkins controller versus agent?
- **What interviewer is testing:** Jenkins architecture. **Short answer:** Controller orchestrates/configures; agents execute builds, preferably ephemeral and isolated.
- **Strong answer:** Keep builds off controller, manage configuration/plugins as code, patch, back up/restore, constrain credentials and untrusted jobs.
- **Follow-up questions:** Executors on controller? **Troubleshooting / architecture angle:** Blast radius. **Memory hook:** Controller schedules; agent works.

### Q148 — Jenkins queue is stuck; troubleshoot.
- **What interviewer is testing:** Operational method. **Short answer:** Read queue reason, then agent labels/capacity/status, cloud provisioning/quota, executor/locks and controller health.
- **Strong answer:** Distinguish demand spike from label mismatch, dead agent, stale lock or controller/plugin issue; restore capacity safely and alarm on queue time.
- **Follow-up questions:** Agent flapping? **Troubleshooting / architecture angle:** Queue item to required label. **Memory hook:** Demand, match, provision, execute.

### Q149 — Rollback versus roll-forward?
- **What interviewer is testing:** Incident decision. **Short answer:** Choose the fastest safe restoration considering change reversibility, data/schema compatibility, confidence and elapsed impact.
- **Strong answer:** Predefine triggers; immutable compute often rolls back, while irreversible data change may require forward fix/restore/reconciliation.
- **Follow-up questions:** Who decides? **Troubleshooting / architecture angle:** Time-box and evidence. **Memory hook:** Restore safely, not ideologically.

### Q150 — How do you discuss limited production experience?
- **What interviewer is testing:** Integrity and transfer of knowledge. **Short answer:** State the boundary clearly, describe hands-on lab/training work, then explain production-safe design, validation and rollback.
- **Strong answer:** “I have not personally owned this failure in production. I reproduced the workflow in a lab; in production I would confirm scope, use read-only evidence, follow change control, stage a reversible mitigation, and involve the service owner.”
- **Follow-up questions:** What did you actually do? **Troubleshooting / architecture angle:** Name concrete commands/artifacts without inventing outcomes. **Memory hook:** Boundary, evidence, approach.

## Practice rule

A senior answer usually follows: requirement -> mechanism -> tradeoff -> failure mode -> evidence -> safe change -> prevention. If experience is lab-based, say so once and continue with technically sound reasoning.
