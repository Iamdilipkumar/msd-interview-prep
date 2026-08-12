# Mock Interview 1 — Technical Storage + AWS

## Format

- Duration: 60–75 minutes.
- Answer core question in 30 seconds, then expand for up to two minutes.
- Do not read prompts below until after answering.
- Score each 0–3: **0 unsafe/wrong, 1 definition, 2 working/trade-offs, 3
  senior path+failure+operations+boundary**. Target ≥48/60 with no unsafe answer.

## 1. Block, file and object

**Core:** Explain them and map EBS, EFS and S3.

- **If vague:** Who owns the filesystem and metadata?
- **If strong:** A vendor proposes S3 for live database files. Challenge it.
- **Evidence probe:** Did you make this decision or is it conceptual?
- **Listen for:** Semantics first; no “block is always fastest.”

## 2. SAN vs NAS

**Core:** Why are SAN and NAS not protocols?

- **If vague:** Place NFS, SMB, iSCSI and FC correctly.
- **If strong:** Draw both I/O paths and shared failure layers.
- **Failure twist:** Two SAN paths exist but one switch fails and both disappear.
- **Listen for:** Architecture vs protocol; correlated path risk.

## 3. LUN path

**Core:** Walk from application to an ONTAP LUN.

- **If vague:** Where are igroup/mapping, multipath and filesystem?
- **If strong:** LUN grew but application capacity did not—why?
- **Failure twist:** Host discovers target but sees no device.
- **Boundary:** Lab path is not production SAN ownership.

## 4. Aggregate, volume, LUN, qtree, SVM

**Core:** Define each without mixing layers.

- **If vague:** Which is physical-capacity layer? Which is logical server?
- **If strong:** Why choose qtree instead of volume, and what independence is lost?
- **Failure twist:** Volume says free space but aggregate is nearly full.
- **Listen for:** Thin-provisioning/headroom awareness.

## 5. NFS latency

**Core:** An NFS application is slow. Investigate.

- **If vague:** Is mount, metadata, read or write slow?
- **If strong:** One client only; array latency normal; retransmits rise.
- **Failure twist:** UID mismatch and packet loss occur together—how isolate?
- **Listen for:** Scope/baseline/path, same-window evidence, safe test.

## 6. SMB access failure

**Core:** User authenticates but cannot open a file.

- **If vague:** Share permission vs filesystem ACL?
- **If strong:** Add split DNS, clock skew and multi-protocol identity mapping.
- **Failure twist:** Only one file is locked; other files work.
- **Listen for:** DNS/time/AD/Kerberos/share/ACL/session layers.

## 7. iSCSI vs FC

**Core:** Compare design and troubleshooting.

- **If vague:** Initiator/target versus HBA/fabric/zoning?
- **If strong:** How prove paths are independent?
- **Failure twist:** LUN is visible through one path only after change.
- **Listen for:** No universal speed claim; mapping and MPIO remain.

## 8. IOPS/latency/throughput

**Core:** Explain relationships using 4 KiB and 1 MiB examples.

- **If vague:** Compute approximate throughput for 10k×4 KiB.
- **If strong:** Why might observed backend bytes differ?
- **Failure twist:** IOPS flat, queue and p99 rise, throughput flat.
- **Listen for:** Saturation knee and workload profile.

## 9. High IOPS, slow app

**Core:** Give at least six plausible layers/causes.

- **If vague:** What if average latency is normal?
- **If strong:** Design the smallest evidence set to prove the layer.
- **Failure twist:** Low CPU and high cache hit ratio.
- **Listen for:** Locks/tail/sync/network/metadata/QoS/queue; no tunnel vision.

## 10. Snapshot vs backup

**Core:** Why is a snapshot not automatically backup?

- **If vague:** What if source array is lost?
- **If strong:** Add application consistency, ransomware and key recovery.
- **Failure twist:** Backup job green; restore cannot authenticate to KMS.
- **Listen for:** Independent/retained/tested recovery.

## 11. SnapMirror failover/failback

**Core:** Walk the conceptual sequence.

- **If vague:** Which copy is authoritative after failover?
- **If strong:** No common snapshot; what risk and options?
- **Failure twist:** Original source returns with stale writes while DR serves users.
- **Boundary:** **REQUIRES REAL EXPERIENCE — DO NOT FABRICATE** a past DR test.
- **Listen for:** Direction, divergence, validation, controlled cutback.

## 12. RAID, HA, multipathing, DR

**Core:** What failure does each cover?

- **If vague:** Which protects deletion/ransomware?
- **If strong:** Find common-mode failures in an apparently redundant design.
- **Failure twist:** One controller and one path are down during maintenance.
- **Listen for:** Layered protection; backup still required.

## 13. RPO/RTO

**Core:** Define and explain achieved versus promised values.

- **If vague:** Is replication schedule the RPO?
- **If strong:** Storage recovered in 5 minutes but app takes 45—did RTO pass?
- **Failure twist:** Replication lag reaches 20 minutes against 15-minute RPO.
- **Listen for:** Last usable copy and user-visible recovery.

## 14. S3/EBS/EFS selection

**Core:** Select for object-native media, database disk and shared Linux content.

- **If vague:** What semantics force each choice?
- **If strong:** Add multi-AZ, encryption, backup and cost.
- **Failure twist:** Application wants RWX and 1 ms p99.
- **Listen for:** Clarify/benchmark; no magic service promise.

## 15. EFS vs FSx for ONTAP

**Core:** Make a decision for on-prem ONTAP NFS migration.

- **If vague:** Which ONTAP features are actually required?
- **If strong:** Compare protocol, HA, tiering, protection, operations and cost.
- **Failure twist:** Multi-protocol SMB/NFS and SnapMirror are mandatory.
- **Boundary:** Résumé FSxN ownership remains unverified unless evidenced.

## 16. FSx for ONTAP architecture

**Core:** Draw file system→SVM→volume→client path.

- **If vague:** Where do VPC routes, endpoints and KMS fit?
- **If strong:** Multi-AZ failure and monitoring design.
- **Failure twist:** NFS endpoint reachable but mount denied.
- **Listen for:** Protocol identity/export plus network, not just SG.

## 17. Large NFS migration

**Core:** Plan discovery through rollback.

- **If vague:** How handle changes during initial copy?
- **If strong:** Validate metadata/identity and calculate cutover window.
- **Failure twist:** Final DataSync has permission errors on 0.1% of files.
- **Boundary:** No invented data size or customer migration.

## 18. IAM/KMS denial

**Core:** IAM policy allows access but encrypted object fails.

- **If vague:** Which other policy layers exist?
- **If strong:** Cross-account role plus endpoint policy and key grant.
- **Failure twist:** Admin works from public path; workload fails through endpoint.
- **Listen for:** Actual principal/action/resource/key, explicit deny, logs.

## 19. VPC/Route 53 storage path

**Core:** Hybrid client cannot resolve/reach storage endpoint.

- **If vague:** Separate DNS from routing.
- **If strong:** Add resolver forwarding, return route, SG/NACL and MTU.
- **Failure twist:** Only one AZ/subnet fails.
- **Listen for:** Client-side test and bidirectional packet path.

## 20. AWS Backup, DataSync, Gateway

**Core:** Give one correct use case for each.

- **If vague:** Which is ongoing hybrid interface?
- **If strong:** Combine all three in a migration/protection architecture.
- **Failure twist:** Gateway cache full while WAN is degraded.
- **Listen for:** Protect vs move vs bridge; recovery/monitoring.

## Score interpretation

- **48–60:** technically ready; polish boundaries and concise delivery.
- **36–47:** review missed paths and repeat tomorrow questions.
- **<36:** prioritize fundamentals, ONTAP objects, performance, DR and AWS choices.
- Any fabricated incident or unsafe “just delete/open/admin” answer = automatic
  remediation regardless of score.
