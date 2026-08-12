# Mock Interview 2 — Senior Architecture + Résumé Cross-Examination

## Format

- Duration: 75 minutes.
- Treat architecture prompts as hypothetical unless a fact is verified.
- Score each 0–3: **0 unsafe/fabricated, 1 generic, 2 sound structure, 3 senior
  trade-offs+evidence+boundary**. Target ≥48/60 and zero fabrication.
- Interviewer should interrupt vague answers and change constraints.

## 1. Opening and fit

**Core:** Introduce yourself for this MSD role in 90 seconds.

- **If vague/tool list:** Give one relevant strength, one real boundary and your learning method.
- **If strong:** Why a senior role when production depth is limited?
- **Evidence probe:** Which claims are verified versus training/lab?
- **Listen for:** Integrity, fundamentals, method, MSD hybrid-storage fit.

## 2. Hybrid storage architecture

**Core:** Design secure hybrid storage for file, block and object workloads.

- **If vague:** Ask for workloads, protocol, latency, data class, RPO/RTO.
- **If strong:** On-prem connectivity fails; which workloads continue?
- **Constraint:** Regulated data, two Regions, cost reduced 20%.
- **Listen for:** Placement, identity/path/key, protection, operations and ADR.

## 3. RPO 15 / RTO 30

**Core:** Design and prove it.

- **If vague:** What is last usable copy and what completes RTO?
- **If strong:** Replication carries corruption; Region unavailable.
- **Constraint:** DNS/identity/database dependencies add 18 minutes.
- **Listen for:** Margin, independent backup, automation, rehearsal and measured recovery.

## 4. Performance architecture

**Core:** Design for latency-sensitive database plus throughput-heavy analytics.

- **If vague:** Ask I/O size/mix/concurrency and SLO percentiles.
- **If strong:** Same backend pool, noisy neighbor appears.
- **Constraint:** Cost prevents dedicated arrays for everything.
- **Listen for:** Separate tiers/QoS, benchmark, capacity, HA/DR and monitoring.

## 5. Storage latency incident

**Core:** App latency jumps 5→40 ms; storage team says array is healthy.

- **If vague:** Where can the extra 35 ms be introduced?
- **If strong:** Only p99, one Kubernetes node, NFS retransmits.
- **Constraint:** Recent network and storage changes occurred simultaneously.
- **Listen for:** Same-window path evidence and one-variable tests.

## 6. Terraform architecture

**Core:** Design repository/state/pipeline for 20 teams and three environments.

- **If vague:** State blast radius, modules, identity and promotion?
- **If strong:** Shared network outputs and breaking module update.
- **Constraint:** Emergency manual change causes drift.
- **Listen for:** Isolated authority/state, stable contracts, policy and reconciliation.

## 7. Terraform failure

**Core:** Apply partly creates storage and state upload appears damaged.

- **If vague:** First action?
- **If strong:** Backend lock remains; resource exists outside trusted state.
- **Constraint:** Production service is using the created resource.
- **Listen for:** Freeze, preserve, inspect reality, precise recovery; no delete/re-run.

## 8. EKS stateful platform

**Core:** Design persistent storage for single-writer DB and shared content.

- **If vague:** EBS vs EFS, topology/access modes?
- **If strong:** AZ failure and backup/application consistency.
- **Constraint:** PVC Pending during node-group replacement.
- **Listen for:** CSI, binding, topology, recovery, operations.

## 9. EKS upgrade cross-examination

**Core:** Résumé says zero-downtime EKS upgrades. Walk through one.

- **If historical detail unsupported:** Stop and state evidence boundary.
- **Technical continuation:** APIs→control plane→add-ons→nodes→workloads→SLI.
- **Deep probe:** Exact versions, PDB, spare capacity, failure and proof graph.
- **Boundary:** **REQUIRES REAL EXPERIENCE — DO NOT FABRICATE.**

## 10. ONTAP architecture cross-examination

**Core:** Résumé says install/configure/manage clustered ONTAP. Draw the estate.

- **If details unsupported:** Identify training/lab scope explicitly.
- **Technical continuation:** cluster/node/HA/aggregate/SVM/LIF/volume/qtree/LUN.
- **Deep probe:** Platform/version, takeover, capacity, monitoring and exact actions.
- **Boundary:** **REQUIRES REAL EXPERIENCE — DO NOT FABRICATE.**

## 11. Tier-3 protocol cross-examination

**Core:** Give one complex NFS/SMB/SAN incident you personally resolved.

- **If none verified:** Say so; do not choose a fictional incident.
- **Technical continuation:** Pick a clearly hypothetical symptom and apply PATHS.
- **Deep probe:** Command/counter that distinguishes network from storage.
- **Boundary:** **REQUIRES REAL EXPERIENCE — DO NOT FABRICATE.**

## 12. SnapMirror cross-examination

**Core:** What exactly did you configure and monitor?

- **If unsupported:** State trained/lab/conceptual boundary.
- **Technical continuation:** source/destination, policy/schedule, lag, common snapshot, DP state.
- **Deep probe:** Failover, reverse-resync, no common snapshot and failback data authority.
- **Boundary:** **REQUIRES REAL EXPERIENCE — DO NOT FABRICATE** past DR.

## 13. C-Mode migration cross-examination

**Core:** Source, target, tool, scale, protocol and your role?

- **If unsupported:** Do not guess 7-Mode or data size.
- **Technical continuation:** discovery→pilot→seed→sync→freeze→cutover→validate→rollback.
- **Deep probe:** Identity/ACL, open files, downtime and acceptance.
- **Boundary:** **REQUIRES REAL EXPERIENCE — DO NOT FABRICATE.**

## 14. Patch/microcode cross-examination

**Core:** Which component/version and change sequence?

- **If unsupported:** Clarify simulator/training versus production.
- **Technical continuation:** compatibility→health/precheck→protection→HA sequence→monitor→postcheck.
- **Deep probe:** Failed takeover, stop criteria, vendor support and rollback feasibility.
- **Boundary:** **REQUIRES REAL EXPERIENCE — DO NOT FABRICATE.**

## 15. FSx for ONTAP cross-examination

**Core:** Draw what you “implemented and managed.”

- **If unsupported:** Label comparison/training/lab precisely.
- **Technical continuation:** file system/HA→VPC/routes→SVM/endpoints→volume→protocol/client.
- **Deep probe:** tiering, throughput/IOPS, KMS, backup/SnapMirror, failure and cost.
- **Boundary:** **REQUIRES REAL EXPERIENCE — DO NOT FABRICATE** production scope.

## 16. Terraform module cross-examination

**Core:** Show module contract and “all environments” consumers.

- **If unsupported:** State retained lab code versus unverified adoption.
- **Technical continuation:** inputs/outputs, stable keys, tests, versions, state, pipeline.
- **Deep probe:** Breaking change, partial apply and state move.
- **Boundary:** **REQUIRES REAL EXPERIENCE — DO NOT FABRICATE** consumer/environment ownership.

## 17. Pipeline metric cross-examination

**Core:** How was 30% faster build/deployment measured?

- **If unsupported:** Say number lacks evidence and should be revised.
- **Technical continuation:** define lead time/duration/frequency, baseline, window, source and change.
- **Deep probe:** Confounders, sample size, team contribution and regressions.
- **Boundary:** **REQUIRES REAL EXPERIENCE — DO NOT FABRICATE.**

## 18. 99.9% uptime cross-examination

**Core:** Define and derive 99.9%.

- **If unsupported:** State boundary; do not say dashboards caused uptime.
- **Technical continuation:** service boundary, SLI numerator/denominator, window, exclusions, source, error budget.
- **Deep probe:** Partial outage, telemetry gap and monitoring causality.
- **Boundary:** **REQUIRES REAL EXPERIENCE — DO NOT FABRICATE.**

## 19. Stakeholder scenario

**Core:** Security rejects your storage design two days before rollout.

- **If vague:** What do you do in the next hour?
- **If strong:** Business refuses delay; risk is material.
- **Constraint:** No authority to accept the risk yourself.
- **Listen for:** Clarify control/evidence, options, escalate, no bypass, update plan/rollback.
- **Experience boundary:** Scenario is hypothetical unless a real example is verified.

## 20. Final challenge

**Core:** What is your largest gap for this role, and what will you do in 30 days?

- **If vague:** Name one technical and one experience gap.
- **If strong:** Give measurable learning evidence without claiming production.
- **Constraint:** Team needs on-call storage support in week two.
- **Listen for:** Honest risk, shadowing/escalation, labs/runbooks, safe scope, review and progress.

## Score interpretation

- **48–60:** credible senior reasoning with safe boundaries.
- **36–47:** technically promising; shorten answers and repair weak cross-exam areas.
- **<36:** do not bluff. Focus on crash course, last-hour comparisons and the
  ten résumé dangers.
- Any invented project, incident, scale, metric, version, stakeholder or outcome
  scores **0** for that question and must be corrected aloud.

## Strong closing behavior

Ask concise questions such as:

1. Which storage platforms/protocols create the largest operational load today?
2. How does the team divide on-prem ONTAP, AWS storage and automation ownership?
3. What would success in the first 90 days look like?
4. How are POCs and production-readiness reviews performed?
5. Which skills should the selected engineer deepen first?
