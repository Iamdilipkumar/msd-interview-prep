# 01 — AWS Foundations

## The senior-level mental model

AWS architecture is constraint management across six axes: availability, durability, security, performance, operability, and cost. Start with workload requirements; do not start by naming services.

```text
Users -> DNS -> edge -> load balancer -> stateless compute -> stateful data
                         |                    |
                    health checks       queues/caches
                         \------ telemetry ------/
                    IAM + encryption + audit everywhere
```

## Scope and fault boundaries

| Boundary | Meaning | Typical design response |
|---|---|---|
| Region | Separate geographic AWS area | Multi-Region only when business RTO/RPO justifies it |
| Availability Zone | Isolated infrastructure within a Region | Run production tiers across at least two AZs |
| Edge location | CloudFront/DNS edge presence | Cache, filter, and terminate close to users |
| Account | Strong billing, policy, and blast-radius boundary | Separate production, nonproduction, security, and shared services |

Multi-AZ addresses an AZ failure. It is not automatically disaster recovery. Multi-Region adds independence but also replication lag, consistency, routing, testing, and cost complexity.

## Shared responsibility

AWS secures **of** the cloud; the customer secures what they configure and run **in** it. Responsibility moves toward AWS as abstraction increases.

| Model | Customer still owns |
|---|---|
| EC2 | Guest OS, patches, application, identity, data, network configuration |
| Containers | Image, application, identities, policies, data, runtime configuration |
| Managed database | Schema, users, queries, data, network/IAM configuration |
| S3 | Data classification, bucket/access policies, retention, encryption choices |

## Well-Architected reasoning

| Pillar | Interview question to ask |
|---|---|
| Operational excellence | How is it deployed, observed, tested, and rolled back? |
| Security | Who can do what, from where, under which conditions, and is it logged? |
| Reliability | What fails, what detects it, and what restores service? |
| Performance efficiency | What bottleneck and scaling dimension matter? |
| Cost optimization | Which capacity is idle, duplicated, or retained unnecessarily? |
| Sustainability | Can utilization and data movement be reduced? |

## Availability, durability, RTO, and RPO

- **Availability:** probability the service is usable when needed.
- **Durability:** probability stored data remains intact.
- **RTO:** maximum acceptable restoration time.
- **RPO:** maximum acceptable data-loss window.

```text
last good copy ----(RPO)---- failure ----(RTO)---- service restored
```

A target is not a design until monitoring, runbooks, ownership, dependency recovery, and regularly timed exercises demonstrate it.

## Service-selection heuristics

| Need | Default thought process |
|---|---|
| Object/blob | S3; examine access pattern, lifecycle, consistency, retrieval time |
| Block device | EBS; examine IOPS, throughput, latency, size, AZ coupling |
| Shared POSIX files | EFS; examine latency mode, throughput, many-client behavior |
| Relational data | RDS/Aurora; examine HA, read scale, transactions, recovery |
| Containers | ECS for AWS-opinionated simplicity; EKS for Kubernetes ecosystem/control needs |
| Decoupling | SQS/SNS/EventBridge; define delivery and idempotency behavior |
| Caching | CloudFront/ElastiCache; define invalidation and stale-data tolerance |

## Architecture interview framework

1. Clarify traffic, data, compliance, regions, RTO/RPO, budget, and team skills.
2. Draw request and data flows; identify trust and failure boundaries.
3. Choose the simplest managed services satisfying requirements.
4. Explain scaling, degradation, backup/restore, DR, security, and observability.
5. State tradeoffs, unknowns, test plan, and migration/rollback path.

## Common traps

- “Multi-AZ means no downtime.” Failover and client reconnection still take time.
- “Auto Scaling fixes performance.” It cannot fix database locks, dependency throttling, or bad queries.
- “Private subnet means secure.” Routes, policies, identities, endpoints, patching, and egress controls still matter.
- “Encrypted means authorized.” Encryption does not replace least privilege.
- “A backup means recoverable.” Restore testing is the evidence.

## 30-second answer

“I design AWS systems from measurable requirements: traffic, data sensitivity, availability, RTO/RPO, operational skill, and cost. I isolate blast radius by account, AZ, and service; prefer managed services where they reduce undifferentiated work; apply least privilege and encryption; and design telemetry, deployment safety, backup, and recovery up front. I then validate failure assumptions through load, restore, and game-day tests.”
