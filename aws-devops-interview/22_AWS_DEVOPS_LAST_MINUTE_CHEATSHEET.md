# 22 — AWS DevOps Last-Minute Cheat Sheet

Designed for a 30–45 minute final pass. Speak the rapid answers aloud; stop reading when recall is strong.

## Top 30 rules

1. Requirements before services: traffic, data, consistency, compliance, RTO/RPO, cost, skills.
2. Multi-AZ handles an AZ failure; Multi-Region is a separate DR design.
3. Availability is service usability; durability is data survival.
4. A backup is proven only by a timed, validated restore.
5. Route chooses; policy permits; application completes; return path matters.
6. Security groups are stateful allow-only; NACLs are stateless ordered allow/deny.
7. Public subnet means route to IGW; resource reachability needs public address and controls.
8. NAT initiates private IPv4 egress; it is not inbound publishing.
9. VPC endpoints give private paths, not automatic authorization.
10. Explicit IAM deny wins; otherwise applicable allow is required.
11. A boundary/SCP is a ceiling, not a grant.
12. Trust policy says who assumes; permission policy says what session does.
13. Prefer federation and temporary roles; never embed access keys.
14. ECS task role is for app; execution role is for ECS agent.
15. EKS Pod Identity/IRSA gives pod identity; do not rely on broad node role.
16. S3 is object, EBS block, EFS shared file.
17. S3 versioning protects versions; lifecycle controls cost; Object Lock controls retention.
18. IOPS × IO size approximates throughput; latency and queue still matter.
19. Multi-AZ DB, read replica, and backup solve different problems.
20. Load balancer distributes; health check admits; Auto Scaling supplies.
21. Readiness routes traffic; liveness restarts; startup protects initialization.
22. Kubernetes requests schedule; limits constrain runtime.
23. A PDB controls voluntary disruption; it does not create availability.
24. Build once; promote the same immutable digest.
25. Database releases use expand/migrate/contract.
26. Terraform state is critical ownership mapping and may contain secrets.
27. One writer: pipeline concurrency plus Terraform state locking.
28. Partial apply means inspect reality/state and re-plan—never retry blindly.
29. Stabilize first, preserve evidence, make smallest reversible change.
30. Never fabricate production experience; state boundary, then engineering method.

## Top troubleshooting flows

### Network: RSPAR

```text
Resolve DNS -> Source/destination -> Path/routes -> Allow SG/NACL/policy -> Return
-> listener/app -> dependency
```

### IAM: PARC

```text
Principal/session -> Action -> Resource ARN -> Context/conditions
-> identity/resource -> boundary/session/SCP -> endpoint -> KMS
```

### Load balancer

```text
DNS -> listener/cert -> rule -> target registration -> health -> SG -> port -> app
502 bad target response | 503 no available healthy target | 504 slow target
```

### Kubernetes

```text
desired state -> events -> pod status -> previous logs -> process/probes
-> Service endpoints -> DNS/policy -> ingress/LB -> dependency -> node/cloud
```

### Database

```text
user latency -> transaction/query -> waits/locks/plan -> connections
-> CPU/memory/cache -> storage -> replica/dependency -> recent change
```

### Terraform

```text
freeze writers -> preserve state/logs -> inspect cloud + state + config
-> fresh plan -> fix root cause -> review -> apply -> verify
```

### Incident

```text
impact/scope -> timeline/change -> evidence -> stabilize -> root cause
-> safe correction -> verify SLO -> prevent/test
```

## Top 20 comparisons

| A | B | Memory line |
|---|---|---|
| Multi-AZ | Multi-Region | AZ HA versus regional DR |
| RTO | RPO | Downtime versus data-loss window |
| SG | NACL | Stateful ENI versus stateless subnet |
| IGW | NAT gateway | Public route versus outbound translation |
| Peering | TGW | Pair versus hub |
| PrivateLink | Peering | Service exposure versus network connection |
| Gateway endpoint | Interface endpoint | S3/Dynamo route versus service ENI |
| IAM role | IAM user | Temporary session versus long-lived identity |
| Trust policy | Permission policy | Who enters versus what they do |
| Boundary | SCP | Identity ceiling versus organization ceiling |
| S3 | EBS | Object API versus block device |
| EBS | EFS | AZ block versus regional shared NFS |
| Versioning | Replication | History versus asynchronous copy |
| Multi-AZ RDS | Read replica | Failover versus read scale |
| ALB | NLB | HTTP L7 versus transport L4 |
| Readiness | Liveness | Receive traffic versus restart |
| HPA | node autoscaling | More pods versus more capacity |
| ECS | EKS | AWS-native simplicity versus Kubernetes ecosystem |
| Rolling | Blue/green | Incremental replacement versus environment switch |
| Artifact | Cache | Trusted deployable output versus speed optimization |

## Top 20 commands and evidence sources

| Need | Tool/concept |
|---|---|
| AWS caller | `aws sts get-caller-identity` |
| API history | CloudTrail event/request ID |
| Modeled VPC path | Reachability Analyzer |
| Network metadata | VPC Flow Logs |
| DNS answer/TTL | `dig <name>` / `nslookup` |
| HTTP/TLS timing | `curl -v` / `curl -w` |
| Listening sockets | `ss -lntp` |
| Host IO | `iostat -xz 1` |
| Host CPU/memory | `top`, `vmstat 1`, process profiler |
| Pod summary | `kubectl get pods -A -o wide` |
| Pod evidence | `kubectl describe pod <pod> -n <ns>` |
| Previous crash | `kubectl logs <pod> -n <ns> --previous` |
| Cluster events | `kubectl get events -n <ns> --sort-by=.metadata.creationTimestamp` |
| Service endpoints | `kubectl get svc,endpoints,endpointslices -n <ns>` |
| RBAC check | `kubectl auth can-i <verb> <resource>` |
| Rollout | `kubectl rollout status deployment/<name> -n <ns>` |
| Terraform format/check | `terraform fmt -check`, `terraform validate` |
| Terraform proposed change | `terraform plan` and reviewed saved plan |
| State inspection | `terraform state list/show`—read first, mutate rarely |
| Deployment proof | Commit + artifact digest + config + SLO metrics |

## Top 15 architecture decisions

1. Account/VPC/cluster boundary by trust and blast radius.
2. AZ count and behavior when one AZ disappears.
3. Regional DR tier tied to measured RTO/RPO.
4. Data access semantics: object, block, file, relational, key-value.
5. Write ownership and consistency for Multi-Region data.
6. Private versus public ingress/egress and inspection path.
7. Workload identity per application; no shared static credentials.
8. Encryption/key ownership and cross-account/Region recovery.
9. Compute choice: EC2, ECS, EKS, or serverless from workload/team needs.
10. Scaling signal that represents demand per unit.
11. Deployment strategy and automated stop/rollback evidence.
12. Backward-compatible schema/configuration evolution.
13. Service SLOs, telemetry correlation, alarm owner, and runbook.
14. Backup immutability/isolation plus full restore test.
15. Cost drivers: idle capacity, requests, storage duration/retrieval, data movement.

## Top traps

- “The resource is in a private subnet, so it is secure.”
- “Multi-AZ guarantees zero downtime.”
- “Replication is a backup.”
- “The IAM policy contains Allow, so it must work.”
- “KMS encryption means anyone with object access can decrypt.”
- “CPU is the universal scaling metric.”
- “A running container is a healthy service.”
- “Kubernetes will always spread replicas across AZs.”
- “PDB protects against node/AZ failure.”
- “Terraform plan cannot expose secrets.”
- “Run Terraform apply again after partial failure.”
- “Force-unlock whenever a lock is inconvenient.”
- “Rollback always reverses a database change.”
- “WAF replaces application authorization.”
- “I did this in production” when experience was training/lab.

## Top 50 rapid questions

1. **Public subnet?** Route to IGW; resource still needs public address and permissions.
2. **SG versus NACL?** Stateful ENI allow versus stateless subnet allow/deny.
3. **Longest prefix?** Most-specific matching route wins.
4. **NAT purpose?** Private IPv4 outbound translation.
5. **Endpoint purpose?** Private path to supported service; policy still applies.
6. **Peering limitation?** Non-transitive and no overlapping CIDRs.
7. **Flow Logs limitation?** Metadata, not payload/app health.
8. **Timeout workflow?** DNS, source, route, policy, listener, return.
9. **DNS change delay?** Recursive/OS/app cache TTL and existing connections.
10. **Explicit deny?** Overrides allows.
11. **Boundary?** Maximum, grants nothing.
12. **SCP?** Organization guardrail, grants nothing.
13. **Role trust?** Who/conditions may assume.
14. **PassRole?** Let a service use a role; restrict service and role.
15. **AccessDenied first fact?** Exact session principal.
16. **EC2 identity?** Instance profile and IMDSv2.
17. **ECS identity?** Task role for app; execution role for agent.
18. **EKS identity?** Pod Identity or IRSA per service account.
19. **S3 consistency?** Strong reads/listings after writes; races still possible.
20. **S3 recovery?** Versioning/retention plus tested restore.
21. **Object Lock?** WORM retention on object versions.
22. **Presigned URL?** Temporary bearer-like signed request.
23. **IO equation?** Throughput ≈ IOPS × IO size.
24. **High IOPS, slow app?** Check latency/queue, locks, CPU and dependencies.
25. **ASG no launch?** Activity reason: quota, IP, capacity, template, KMS/IAM.
26. **ALB 502?** Invalid/reset target response.
27. **ALB 503?** No available healthy targets.
28. **ALB 504?** Target response timeout.
29. **Multi-AZ DB?** Availability/failover.
30. **Read replica?** Asynchronous read scaling/DR option.
31. **PITR result?** New DB needing validation/cutover.
32. **Connection storm defense?** Bounded global pools, proxy where fit, backoff.
33. **EKS Pending first check?** Pod events.
34. **PVC Pending?** Class, CSI/IAM, binding mode, topology, quota.
35. **CrashLoop?** Previous logs and exit reason.
36. **Readiness?** Whether target should receive traffic.
37. **Liveness?** Whether container should restart.
38. **Requests?** Scheduling reservation.
39. **Limits?** Runtime constraint.
40. **PDB?** Voluntary-disruption constraint only.
41. **EKS upgrade order?** Preflight; control plane; add-ons; nodes; workloads verify.
42. **Build once?** Promote exact tested digest.
43. **OIDC in CI?** Exchange trusted claims for short-lived role.
44. **Terraform state?** Resource ownership/address mapping; sensitive.
45. **Current S3 lock?** Native `use_lockfile`; verify Terraform version.
46. **Partial apply?** Freeze, inspect reality/state, fix, re-plan.
47. **Unexpected destroy?** Stop; inspect address/force-new/state before approval.
48. **Secret marked sensitive?** Redacted, potentially still stored in state/plan.
49. **First incident action?** Define impact and stabilize with reversible move.
50. **Limited experience?** State boundary honestly, then give safe technical approach.

## 60-second opening framework

“My focus is reliable, secure infrastructure and repeatable delivery. I reason from requirements and failure modes, then use AWS networking, identity, compute, storage, containers, Terraform, and CI/CD to build an operable solution. I emphasize least privilege, immutable artifacts, observability, tested recovery, and reversible change. Where my experience is lab or training rather than production ownership, I state that clearly and explain how I would validate and execute safely in production.”

## Final answer pattern

```text
Clarify requirement -> state mechanism -> explain tradeoff
-> name failure mode -> show evidence/troubleshooting
-> safe mitigation/rollback -> prevention and test
```
