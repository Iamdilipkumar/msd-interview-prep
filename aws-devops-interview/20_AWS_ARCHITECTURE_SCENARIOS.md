# 20 — AWS Architecture Scenarios

## Interview framework

Before drawing, clarify users/traffic, data model and growth, consistency, compliance, dependencies, regions, RTO/RPO, budget, and team skills. Then show request/data flow, failure boundaries, security, observability, deployment, recovery, and cost. These 25 scenarios provide starting designs—not universal answers.

### AS01 — Highly available three-tier application

**Design:** Route 53 -> CloudFront/WAF -> ALB across AZs -> stateless compute across AZs -> Multi-AZ RDS/Aurora; S3 for objects; queues for asynchronous work. **Tradeoffs:** Managed HA costs more; database remains a scaling/failure focus. **Prove:** AZ-failure, restore, load, and rollout tests.

### AS02 — Global read-heavy web platform

**Design:** CloudFront caching, regional API deployments, globally suitable data replication, health-based/latency routing. **Tradeoffs:** Cache staleness and replicated-write consistency; Multi-Region operations. **Decision:** Define where writes occur before selecting the database.

### AS03 — RPO 15 minutes, RTO 30 minutes DR

**Design:** Warm standby with asynchronous cross-Region data copies/replication, IaC-provisioned regional dependencies, reduced live capacity, tested promotion and DNS/traffic shift. **Tradeoffs:** More cost than backup/restore; lower complexity than active/active. **Prove:** Timed full-stack game day including failback and data reconciliation.

### AS04 — Static website and API

**Design:** Private S3 origin through CloudFront OAC; WAF; API Gateway/Lambda or ALB/container API; separate cache behaviors. **Tradeoffs:** Serverless simplicity versus latency/runtime limits and cost at sustained scale.

### AS05 — Secure multi-account platform

**Design:** Organization OUs/SCP guardrails; separate prod/nonprod/security/log archive/shared services accounts; Identity Center federation; centralized CloudTrail/Config/security findings; scoped network hub. **Tradeoffs:** Governance improves blast radius but adds vending, policy, and cross-account complexity.

### AS06 — Hybrid enterprise connectivity

**Design:** Redundant Direct Connect connections/locations into Transit Gateway plus redundant VPN backup; segmented TGW route tables; centralized DNS Resolver and monitored path. **Tradeoffs:** Predictability and private routing versus cost/lead time; carefully control asymmetric routes.

### AS07 — Large on-premises migration

**Design:** Discover dependencies and data; establish landing zone/connectivity; classify rehost/replatform/refactor/retire; seed bulk data with online/offline transfer; continuously synchronize; wave-based cutover and rollback. **Tradeoffs:** Longer coexistence costs versus lower cutover risk. Validate data and application behavior, not just byte counts.

### AS08 — Multi-tenant SaaS

**Design:** Tenant identity propagated through edge/API/service/data; pooled, siloed, or hybrid isolation based on risk; per-tenant quotas, encryption context, logs, and cost attribution. **Tradeoffs:** Stronger isolation costs more and reduces pooling efficiency. Prevent noisy-neighbor and authorization-by-ID bugs.

### AS09 — Event-driven order processing

**Design:** API writes durable intent; queue/event bus decouples workers; idempotency keys; DLQ/redrive; transactional outbox where dual-write consistency matters. **Tradeoffs:** Eventual consistency and duplicate delivery require explicit semantics and observability.

### AS10 — EKS platform for many teams

**Design:** Account/cluster boundaries by trust, managed node groups/Fargate as appropriate, Pod Identity, GitOps/deployment standards, ingress/DNS/cert controllers, policy/admission, centralized telemetry and cost allocation. **Tradeoffs:** Shared clusters improve utilization but increase multitenancy blast radius and platform complexity.

### AS11 — ECS microservices platform

**Design:** ECS services on Fargate/capacity providers, ALB/service discovery, task roles, private ECR/endpoints, autoscaling, circuit-breaker deployments, centralized logs/traces. **Tradeoffs:** AWS-native simplicity versus Kubernetes ecosystem/portability.

### AS12 — Database read scaling

**Design:** Optimize queries/indexes first; use replicas/reader endpoint, cache, and read/write routing; monitor lag and define consistency-sensitive reads. **Tradeoffs:** Replicas add cost and stale-read behavior; cache invalidation adds complexity.

### AS13 — Zero-trust CI/CD to AWS

**Design:** GitHub/Jenkins federates to environment-specific deploy roles with short sessions; immutable signed artifacts; protected approvals; policy checks; canary; SLO rollback. **Tradeoffs:** Stronger controls slow exceptional changes unless break-glass is designed and audited.

### AS14 — Terraform at organizational scale

**Design:** Small root modules/states by lifecycle; versioned modules; S3 versioned encrypted backend with native locking; CI plan/apply separation; account-specific roles; policy/test gates; drift detection. **Tradeoffs:** More states require orchestration but reduce lock contention and blast radius.

### AS15 — Centralized observability

**Design:** Standard telemetry schema/correlation, account-local collection, cross-account central analysis/archive, SLO dashboards and owned alerts, restricted log access and retention. **Tradeoffs:** Central visibility versus transfer/ingest cost and sensitive-data concentration.

### AS16 — Secure file ingestion

**Design:** Short-lived presigned S3 upload, quarantine prefix/bucket, event-triggered validation/malware workflow, metadata state machine, immutable audit, promote clean objects. **Tradeoffs:** Asynchronous user experience and scanner cost; never serve quarantine objects.

### AS17 — Cost-aware batch analytics

**Design:** S3 data lake with partitioned formats/lifecycle, event/scheduled orchestration, Spot-capable batch compute, checkpoints/idempotency, catalog and least privilege. **Tradeoffs:** Spot/queue delay and data-format complexity for substantial savings.

### AS18 — Private API consumed across accounts

**Design:** Producer NLB/service exposed via PrivateLink; consumer interface endpoints; private DNS and explicit IAM/application auth. **Tradeoffs:** Service-level isolation and no network mesh, but endpoint costs and one-way consumer initiation constraints.

### AS19 — Active/active regional API

**Design:** Independently deployable regional stacks, global routing, region-local dependencies, globally appropriate data model, idempotent writes/conflict strategy. **Tradeoffs:** Best regional resilience and latency, highest data/operational complexity. Do not promise active/active without resolving write ownership.

### AS20 — Regulated immutable backup

**Design:** Classified assets, cross-account backup vault, logically isolated administration, KMS governance, retention lock/Object Lock where appropriate, cross-Region copy, restore environment and evidence. **Tradeoffs:** Immutability limits emergency deletion and raises retention cost; legal requirements drive policy.

### AS21 — Blue/green EKS migration

**Design:** Build new cluster with compatible APIs/add-ons/identity/storage, continuously deploy/test workloads, migrate state separately, shift weighted traffic, observe, retain rollback window. **Tradeoffs:** Strong rollback/isolation versus duplicate capacity and stateful migration complexity.

### AS22 — Platform degraded by dependency outage

**Design:** Define critical dependencies; timeouts shorter than upstream budgets; bounded retries with jitter; circuit breakers, queues, caches/stale reads, bulkheads, load shedding, and honest degraded-mode UX. **Tradeoffs:** Staleness/partial functionality versus total outage. Exercise the exact failure mode.

### AS23 — Secure secrets architecture

**Design:** Workloads use Pod Identity, task roles or instance profiles to retrieve narrowly scoped Secrets Manager values at runtime; KMS key policy separates administration/use; rotation supports overlapping versions and client refresh; CloudTrail/detection monitors access. **Tradeoffs:** Runtime retrieval and rotation improve control but add KMS/secret-service dependency, caching and failure-mode design. Never bake values into images, Terraform state or CI logs.

### AS24 — RDS high-availability architecture

**Design:** Private Multi-AZ RDS/Aurora across appropriate AZs, DNS endpoint through bounded connection pools/RDS Proxy where suitable, tested automated backup/PITR, read replicas only for read scale/DR needs, encrypted secrets and SLO monitoring. **Tradeoffs:** Higher availability adds standby/replica cost and failover still interrupts connections; client retry/idempotency and timed exercises determine actual RTO.

### AS25 — Private application with secure S3

**Design:** Internal or controlled ingress reaches private compute across AZs; workloads use roles and an S3 gateway endpoint; bucket policy limits intended principals/path, Block Public Access remains on, SSE-KMS/versioning/retention protect data, and CloudTrail provides audit evidence. **Tradeoffs:** Private endpoints reduce internet/NAT exposure but add policy and DNS/routing dependencies; KMS adds control, cost and recovery requirements.

## Strong closing statement

“This is the initial design. I would validate its assumptions with threat modeling, load and failure tests, a measured cost model, restore and failover exercises, and operational review. The final service choices depend on the clarified consistency, RTO/RPO, compliance, and team constraints.”
