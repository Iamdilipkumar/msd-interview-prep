# Last 60 Minutes — MSD Interview

## Use the hour

| Minutes | Action |
|---:|---|
| 0–10 | Say the introduction; review résumé-danger questions and boundaries |
| 10–25 | Storage/ONTAP facts, comparisons and paths |
| 25–35 | Performance, DR and troubleshooting flows |
| 35–45 | AWS storage/IAM/VPC and Terraform recovery |
| 45–53 | EKS/PVC, CI/CD and observability |
| 53–60 | Answer the top oral drill; breathe; stop adding new facts |

## Top 50 facts

1. Block = raw chunks; host owns filesystem.
2. File = paths/directories; file server owns filesystem.
3. Object = key + data + metadata through API.
4. S3 is not a normal POSIX filesystem.
5. EBS=block; EFS=managed NFS; S3=object.
6. SAN is a block architecture, not “fast storage.”
7. NAS is file architecture; it is not synonymous with NFS.
8. NFS and SMB are file protocols.
9. iSCSI carries SCSI block traffic over IP.
10. FC is a block fabric technology, not a filesystem.
11. Aggregate = ONTAP RAID-protected physical capacity/placement layer.
12. Volume = logical data container backed by an aggregate.
13. LUN = block object inside a volume, mapped to hosts.
14. SVM = logical storage server; data LIF = access endpoint.
15. Qtree = subdivision within a NAS volume, often for quota/security/organization.
16. Host owns filesystem on a SAN LUN; storage owns it for NAS.
17. Snapshot is not automatically independent backup.
18. Crash-consistent is not automatically application-consistent.
19. SnapMirror replicates according to relationship/policy; version matters.
20. SnapVault is retention-oriented historical terminology; verify ONTAP version/policy.
21. Failover makes recovery copy authoritative; failback must reverse data direction safely.
22. Resync direction can discard changes—protect known-good data first.
23. RAID protects media failures; it is not backup.
24. HA protects local service; DR handles broader failure.
25. Multipathing protects paths only when paths are truly independent.
26. IOPS=how many; latency=how long; throughput=how much.
27. Throughput ≈ IOPS × average I/O size.
28. 10k×4 KiB≈39 MiB/s; 1k×1 MiB≈1,000 MiB/s.
29. Workload = size + random/sequential + read/write + concurrency + cache state.
30. High IOPS is demand, not automatic health.
31. High latency is not automatic disk failure.
32. Queue depth can be symptom, tuning input or contributor.
33. p99 latency: 99% at/below it; slowest 1% worse.
34. Burst is temporary; baseline is normal sustainable behavior.
35. Backup=retained recovery copy; replication=current second copy; DR=end-to-end service recovery.
36. RPO=data-loss window; RTO=usable-service recovery time.
37. A 15-minute schedule does not prove achieved 15-minute RPO.
38. Storage failover time alone does not prove application RTO.
39. EFS is simple shared Linux NFS; FSxN adds ONTAP/multi-protocol features.
40. DataSync moves/copies; Storage Gateway provides an ongoing hybrid bridge.
41. Route 53=name; VPC=path; IAM=permission; KMS=key permission.
42. Explicit deny wins IAM evaluation.
43. IAM allow does not override KMS key-policy/grant denial.
44. Terraform config says desired; state maps real resource identity.
45. Remote state needs security, backup and locking.
46. Terraform does not automatically roll back a partial apply.
47. Never delete state or force-unlock blindly.
48. PVC asks; StorageClass provisions; PV represents; CSI connects.
49. Prometheus collects/queries metrics; Grafana visualizes/contextualizes.
50. Lab experience is genuine lab experience—not production ownership.

## Top 20 comparisons

| # | A vs B | One-line distinction |
|---:|---|---|
| 1 | Block vs file | Host-owned filesystem vs server-owned shared filesystem |
| 2 | File vs object | Mount/path/locking vs API/key/metadata |
| 3 | SAN vs NAS | Block network architecture vs file service architecture |
| 4 | NFS vs NAS | Protocol vs architecture/service |
| 5 | iSCSI vs FC | SCSI over IP/Ethernet vs SCSI over FC fabric |
| 6 | LUN vs volume | Host-presented block object vs logical container |
| 7 | Volume vs aggregate | Logical data layer vs physical RAID-capacity layer |
| 8 | Qtree vs volume | Lightweight subdivision vs independent logical container/lifecycle |
| 9 | SVM vs node | Logical storage server vs physical/virtual controller member |
| 10 | Snapshot vs backup | Source-dependent point-in-time recovery vs retained recovery copy |
| 11 | SnapMirror vs SnapVault | Replication/protection family vs retention-oriented historical workflow; verify version |
| 12 | RAID vs backup | Stay running through media failure vs recover older/independent copy |
| 13 | HA vs DR | Local continuity vs broader recovery |
| 14 | Sync vs async replication | Wait/coupling/near-zero RPO vs lag window/flexibility |
| 15 | RPO vs RTO | Tolerable data loss vs tolerable recovery time |
| 16 | EBS vs EFS | Block/AZ attachment vs shared managed NFS |
| 17 | EFS vs FSxN | Straightforward NFS vs ONTAP/multi-protocol/data-management features |
| 18 | DataSync vs Gateway | Transfer/migration vs ongoing local hybrid interface |
| 19 | `count` vs `for_each` | Index identity vs stable key identity |
| 20 | Prometheus vs Grafana | Metric collection/query/rules vs visualization/context |

## Top 20 commands / concepts

Commands are evidence collectors. Do not run changes in an interview or unknown
environment.

| # | Command/concept | What it tells you |
|---:|---|---|
| 1 | `lsblk -f` | Block topology, filesystem and mount metadata |
| 2 | `findmnt` | Actual mount source, target, type and options |
| 3 | `df -hT` | Mounted filesystem capacity/type |
| 4 | `df -i` | Inode exhaustion |
| 5 | `du -xsh PATH` | File usage within one filesystem; compare carefully with `df` |
| 6 | `mount` / `/proc/mounts` | Effective mounts/options |
| 7 | `iostat -xz 1` | Device throughput, utilization, queue/latency indicators; interpret by platform |
| 8 | `nfsstat -c` | NFS client/RPC behavior and retransmission clues |
| 9 | `ss -tn` | TCP connection state |
| 10 | `ip route get ADDRESS` | Chosen route/path |
| 11 | `dig NAME` / `getent hosts NAME` | DNS answer / OS resolver view |
| 12 | `multipath -ll` | Multipath device/path state; requires configured host privileges |
| 13 | `fio` | Controlled workload generation—not proof of production performance |
| 14 | Percentile | Tail distribution; use p95/p99, not average alone |
| 15 | Baseline | Known normal comparison for same workload/time context |
| 16 | `terraform plan` | Proposed diff, not guaranteed future execution |
| 17 | `terraform state pull` | Read state copy; protect sensitive output |
| 18 | `terraform force-unlock LOCK_ID` | High-risk stale-lock recovery only after proving no active owner |
| 19 | `kubectl describe pvc NAME` | PVC events, class, binding/provisioning errors |
| 20 | `kubectl describe pod NAME` | Scheduling, attach/mount and container events |

## Top 15 architecture decisions

1. Access model: block, file or object?
2. Protocol: NFS, SMB, iSCSI, FC or API—and client compatibility?
3. Workload: I/O size, mix, pattern, concurrency, latency/throughput SLO?
4. Sharing/locking: single host, cluster-coordinated or many clients?
5. Failure domains: path, controller, rack, AZ, site, Region?
6. HA mechanism and how clients behave during failure?
7. Backup/replication combination and corruption/ransomware isolation?
8. RPO/RTO and measurable dependency recovery order?
9. Identity: UID/GID, AD/Kerberos, IAM roles, cross-account?
10. Encryption/key ownership and recovery if the key/owner fails?
11. Private connectivity, DNS, route and bandwidth/latency?
12. Capacity/headroom, thin provisioning, growth and cost?
13. Observability: user SLI plus every I/O-path layer?
14. Automation/state/change gates and partial-failure recovery?
15. Migration/cutover/rollback/validation and later decommission?

## Top 15 troubleshooting flows

1. **Universal:** symptom→scope→timeline→change→baseline→path→measure→hypothesis→safe test→mitigate→validate→RCA→prevent.
2. **Latency:** app→OS/filesystem→client/driver→network/fabric→frontend/QoS→controller/cache→pool/media.
3. **NFS slow:** operation type→client comparison→DNS/route/loss→mount options/RPC→UID/export/locks→server/backend.
4. **NFS denied:** endpoint→export policy→client identity→root squash→UID/GID→mode/ACL.
5. **SMB auth:** DNS/time→AD/Kerberos→account/SPN→share ACL→filesystem ACL→session/lock.
6. **LUN absent:** discovery/fabric→target→host identity→mapping→rescan→multipath→device.
7. **Path degraded:** HBA/NIC→switch/fabric/network→target port→controller ownership→ALUA/policy.
8. **Capacity full:** identify filesystem/inode/quota/volume/snapshot/pool→stop growth→safe headroom→validate→forecast fix.
9. **Replication lag:** last good point→change rate→schedule/state→network→load/capacity→common snapshot→RPO risk.
10. **Backup failure:** selected source→policy/schedule→agent/service→network→permission/KMS→vault/capacity→restore test.
11. **AWS access:** caller→DNS→route/SG/NACL/endpoint→IAM/resource deny→KMS→CloudTrail/service log.
12. **Terraform partial:** freeze→error→state+reality inventory→fix cause→fresh plan→reconcile→validate.
13. **State corruption:** stop→back up→trusted version→compare reality→precise recovery→plan→validate.
14. **PVC Pending:** events→class→provisioner/CSI→IAM/quota→capacity/access→topology/scheduler.
15. **Pipeline failure:** trigger→workflow→runner→dependency/artifact→identity/gate→target state→safe recovery.

## Top 10 résumé-danger questions

1. Which ONTAP platform/version/topology did you personally manage?
2. Describe one real Tier-3 NFS, SMB or SAN incident with evidence.
3. What exactly did you configure for SnapMirror/SnapVault, and how did failback work?
4. Which aggregate/RAID operations did you perform on physical systems?
5. What source, target, data size, tool and role applied to the C-Mode migration?
6. Which patch/microcode component/version did you upgrade, and what were stop criteria?
7. Draw the production FSx for ONTAP environment you “implemented and managed.”
8. Show the Terraform modules/consumers/state layout “across all environments.”
9. Prove the EKS upgrade had zero downtime—versions, SLI and graph.
10. Derive every 25%, 30%, 40% and 99.9% résumé number.

If evidence is absent: state the boundary. **REQUIRES REAL EXPERIENCE — DO NOT
FABRICATE.**

## Common bad answers to avoid

| Bad answer | Better behavior |
|---|---|
| “SAN is faster than NAS.” | Describe access model/workload; performance is design-dependent. |
| “NFS is NAS.” | NFS is a protocol; NAS is architecture/service. |
| “S3 is a filesystem.” | Object API/key semantics; adapters differ. |
| “Snapshot is backup.” | Explain dependency and layered protection. |
| “RAID means no data loss.” | Defined media tolerance only; backup/DR remain. |
| “High IOPS means healthy.” | Correlate workload, latency, throughput, queue and SLO. |
| “Storage is slow.” | Identify where latency enters the path. |
| “Just increase queue depth.” | Find saturation knee and constrained layer. |
| “Just break/resync SnapMirror.” | Confirm authority/direction; resync can discard changes. |
| “Terraform will roll back.” | Partial apply needs inspection/reconciliation. |
| “Delete state and rerun.” | Freeze, preserve, compare and recover precisely. |
| “Open the SG/admin policy to test.” | Test narrowly with logs and revert safely. |
| “PVC Pending means delete PVC.” | Read events/class/CSI/topology first. |
| “Prometheus ensured 99.9%.” | Require SLI formula, period, source and causality. |
| Invented personal incident | State boundary; provide hypothetical engineering method. |

## Experience-boundary language

Choose the true form:

- “I trained on this and understand the architecture; I have not operated it at
  large production scale.”
- “I built and broke this in a lab. The production differences I would plan for
  are availability, security, monitoring, change control and recovery.”
- “I have not personally handled that incident. My investigation would follow
  the I/O path and test these hypotheses using these signals.”
- “I want to be exact about my role: [verified action]. The broader team outcome
  was not solely my work.”
- “That résumé result is not evidence-backed in the material I have today; I
  should not claim the number as verified.”

Do not over-apologize. Boundary → knowledge → practice → production delta →
approach.

## Interview opening / introduction framework

Use facts only; 60–90 seconds:

1. **Present:** “My background spans IT, with focused training and lab practice
   across enterprise storage, AWS and DevOps.”
2. **Relevant strengths:** storage fundamentals/ONTAP concepts, AWS storage,
   Terraform/automation, CI/CD and Kubernetes—only at truthful evidence levels.
3. **Method:** “I reason end to end, validate with labs and evidence, and focus
   on safe automation, recoverability and clear communication.”
4. **Boundary:** “I’m careful to distinguish production work from training and
   lab practice; I won’t overstate ownership.”
5. **MSD fit:** connect to hybrid storage lifecycle, resilience, POCs,
   security/compliance and learning.

Avoid reciting every tool, employer/customer story or unsupported metric.

## Final 10 oral answers

1. Block/file/object and EBS/EFS/S3.
2. SAN/NAS and NFS/SMB/iSCSI/FC.
3. Aggregate/volume/LUN/SVM/qtree.
4. SnapMirror failover/failback and warning.
5. High IOPS but slow application.
6. RPO 15/RTO 30 hybrid design.
7. EFS vs FSx for ONTAP.
8. Terraform state corruption and partial apply.
9. EKS upgrade and PVC Pending.
10. Truthful response to an unverified résumé claim.
