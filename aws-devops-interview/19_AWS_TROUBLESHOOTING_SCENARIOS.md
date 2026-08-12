# 19 — AWS Troubleshooting Scenarios

## Universal incident loop

```text
scope/impact -> timeline/change -> layer hypothesis -> evidence -> stabilize
             -> smallest safe correction -> verify -> prevent
```

Preserve evidence, use read-only checks first, change one variable at a time, and name rollback criteria. The following 47 scenarios are deliberately compact; practice expanding each aloud.

## Network and edge

### TS01 — Private EC2 cannot reach the internet

- **Symptom:** Outbound HTTPS times out. **Likely layers:** DNS, subnet route, NAT, NACL/SG, host.
- **Check order and tools / signals:** Resolve DNS; inspect source subnet route; NAT state/metrics; Flow Logs/Reachability Analyzer; `curl`; host proxy/firewall.
- **Likely root causes:** Route points to wrong/AZ-failed NAT, NAT absent, NACL blocks ephemeral return, DNS failure.
- **Safe fix:** Correct the explicit route/control; use service endpoint where appropriate. **Prevention:** Per-AZ egress design, alarms, route policy tests.

### TS02 — Public EC2 is unreachable

- **Symptom:** Connection timeout. **Likely layers:** addressing, IGW/route, SG, NACL, OS/service.
- **Check order and tools / signals:** Confirm public address and route; SG source/port; NACL both ways; Flow Logs; status checks; listener/host firewall.
- **Likely root causes:** No public IP, wrong SG, process bound to localhost, return traffic blocked.
- **Safe fix:** Prefer Session Manager; narrowly correct access. **Prevention:** Avoid public admin ports, test IaC controls.

### TS03 — Inter-VPC traffic fails

- **Symptom:** Connected VPC workloads time out. **Likely layers:** CIDR, peering/TGW, routes, policies, DNS.
- **Check order and tools / signals:** Check overlap and attachment state; forward/return routes; TGW route domains; SG/NACL; Flow Logs; Resolver.
- **Likely root causes:** Missing return route, non-transitive peering assumption, wrong TGW propagation.
- **Safe fix:** Add precise symmetric route. **Prevention:** Route-domain tests and IPAM.

### TS04 — Hybrid VPN is intermittent

- **Symptom:** Loss/reconnects to on-premises. **Likely layers:** tunnels/BGP, MTU, routes, firewall, application.
- **Check order and tools / signals:** Tunnel/BGP metrics and logs; route preference; both tunnels; packet size/DF tests; device logs.
- **Likely root causes:** One tunnel unused/broken, asymmetric route, MTU, rekey, saturated internet path.
- **Safe fix:** Restore redundant routing or shift path. **Prevention:** Exercise both tunnels, Direct Connect plus VPN where justified.

### TS05 — NAT gateway connections fail under load

- **Symptom:** New outbound connections time out during peaks. **Likely layers:** application reuse, ports, NAT, destination.
- **Check order and tools / signals:** NAT error/connection metrics; five-tuples/Flow Logs; client retry/pooling; destination limits.
- **Likely root causes:** Port exhaustion to one destination, connection leak, retry storm.
- **Safe fix:** Reuse connections, spread destinations/NAT IP capacity, rate limit. **Prevention:** Capacity alarms/load tests/endpoints.

### TS06 — VPC endpoint returns AccessDenied

- **Symptom:** Private connectivity works but API is denied. **Likely layers:** DNS/path, endpoint policy, IAM, resource policy, KMS.
- **Check order and tools / signals:** Confirm endpoint DNS; CloudTrail principal/action/resource; simulate policies; inspect endpoint/bucket/key policy.
- **Likely root causes:** Endpoint-policy deny, wrong principal, missing KMS permission.
- **Safe fix:** Narrowly allow required tuple. **Prevention:** Contract tests from workload identity.

### TS07 — Route 53 change appears inconsistent

- **Symptom:** Some clients see old target. **Likely layers:** authoritative DNS, recursive/OS/app cache, connections.
- **Check order and tools / signals:** `dig` via multiple resolvers; record/TTL/delegation; app cache; persistent connection state.
- **Likely root causes:** Cached TTL/negative response, wrong hosted zone, connection reuse.
- **Safe fix:** Validate new target and allow cache expiry/controlled traffic shift. **Prevention:** Lower TTL before planned change.

### TS08 — ALB returns 502

- **Symptom:** Bad Gateway. **Likely layers:** listener/target protocol, app process, connection/TLS.
- **Check order and tools / signals:** ALB access logs/metrics; target health; app logs; direct target request; protocol/port/timeouts.
- **Likely root causes:** Target reset/malformed response, TLS mismatch, app crash.
- **Safe fix:** Roll back bad version or correct protocol. **Prevention:** contract/smoke tests and canary alarms.

### TS09 — ALB returns 503

- **Symptom:** Service unavailable. **Likely layers:** target registration/health, capacity, routing rule.
- **Check order and tools / signals:** HealthyHostCount; service/ASG events; target group/AZ; rules; SG and health path.
- **Likely root causes:** No healthy targets, failed deployment, exhausted capacity.
- **Safe fix:** Restore last known-good targets/capacity. **Prevention:** minimum healthy gates and pre-traffic checks.

### TS10 — ALB returns 504

- **Symptom:** Gateway timeout. **Likely layers:** target, dependency, database, timeout chain.
- **Check order and tools / signals:** TargetResponseTime/access logs; traces; app thread/connection pools; DB waits; downstream latency.
- **Likely root causes:** Slow query/dependency, pool starvation, mismatched timeouts.
- **Safe fix:** Shed load/rollback/scale bottleneck. **Prevention:** timeout budgets, load tests, bulkheads.

### TS11 — CloudFront serves stale content

- **Symptom:** Old asset despite origin update. **Likely layers:** behavior, cache key, TTL, invalidation, origin.
- **Check order and tools / signals:** Response/cache headers; behavior order; key inputs; invalidation; origin version.
- **Likely root causes:** Reused object key with long TTL, wrong behavior, cached error.
- **Safe fix:** Publish versioned key or targeted invalidation. **Prevention:** content-hashed assets.

### TS12 — CloudFront/WAF returns 403

- **Symptom:** Viewer denied. **Likely layers:** WAF, distribution, signed access, origin policy.
- **Check order and tools / signals:** WAF/CloudFront logs and request ID; rule match; hostname/cert; OAC bucket policy; origin response.
- **Likely root causes:** WAF false positive, invalid signature, origin access denial.
- **Safe fix:** Scoped WAF exclusion or policy repair. **Prevention:** count-mode rollout and synthetic tests.

## Identity, data, and compute

### TS13 — AssumeRole is denied

- **Symptom:** STS AccessDenied. **Likely layers:** caller permission, role trust, SCP/boundary/session, conditions.
- **Check order and tools / signals:** Caller identity; CloudTrail; trust principal/conditions; caller `sts:AssumeRole`; policy simulator.
- **Likely root causes:** Wrong session principal, subject/external ID mismatch, SCP deny.
- **Safe fix:** Correct exact trust/caller rule. **Prevention:** automated federation tests.

### TS14 — `iam:PassRole` failure

- **Symptom:** Resource creation cannot pass execution role. **Likely layers:** deploy identity and target service.
- **Check order and tools / signals:** CloudTrail action/resource; deploy policy; role ARN; `iam:PassedToService` condition.
- **Likely root causes:** Missing PassRole or role outside allowed path.
- **Safe fix:** Allow only required role/service. **Prevention:** standardized deploy-role patterns.

### TS15 — KMS decrypt denied

- **Symptom:** Encrypted object/secret cannot be read. **Likely layers:** IAM, key policy/grant, region/key state, context.
- **Check order and tools / signals:** Exact principal/key ARN; CloudTrail; key policy/grants; SCP/endpoint; encryption context.
- **Likely root causes:** Key policy excludes role, wrong Region, context mismatch, disabled key.
- **Safe fix:** Narrow policy/grant correction. **Prevention:** integration tests and key-owner runbook.

### TS16 — S3 GetObject denied but List works

- **Symptom:** Keys list; object read fails. **Likely layers:** object ARN policy, bucket policy, ownership, KMS.
- **Check order and tools / signals:** Caller/action/key; CloudTrail data event; object policy/ownership; KMS authorization.
- **Likely root causes:** Only bucket ARN allowed, explicit resource deny, missing decrypt.
- **Safe fix:** Correct object resource/KMS permission. **Prevention:** least-privilege tests for each API.

### TS17 — S3 upload is slow

- **Symptom:** Low throughput/high retries. **Likely layers:** client, path, request shape, S3/KMS.
- **Check order and tools / signals:** Object size/concurrency; multipart use; latency/status/retries; NAT/endpoint; CPU/network; KMS throttle.
- **Likely root causes:** Serial upload, distant path, tiny requests, KMS throttling.
- **Safe fix:** Multipart/parallelism within limits or path correction. **Prevention:** representative transfer benchmarks.

### TS18 — EC2 launch fails

- **Symptom:** Pending then terminated or capacity error. **Likely layers:** quota/capacity, launch template, KMS/IAM, subnet.
- **Check order and tools / signals:** ASG/EC2 reason; quotas; instance flexibility; subnet IPs; AMI/snapshot/KMS permissions.
- **Likely root causes:** AZ capacity, quota, invalid template, encrypted AMI key denial.
- **Safe fix:** Diversify types/AZs or correct template policy. **Prevention:** capacity strategy and launch canaries.

### TS19 — EC2 bootstrapping fails

- **Symptom:** Instance healthy but app absent. **Likely layers:** cloud-init, network, IAM, package/service.
- **Check order and tools / signals:** Console/cloud-init logs; user-data exit; DNS/NAT; role; filesystem; systemd logs.
- **Likely root causes:** Non-idempotent script, repo unreachable, secret denied.
- **Safe fix:** Correct image/script and replace instance. **Prevention:** bake/test immutable AMIs.

### TS20 — EBS latency spikes

- **Symptom:** Application stalls with storage queue. **Likely layers:** workload IO, volume, instance, filesystem.
- **Check order and tools / signals:** EBS latency/queue/IOPS/throughput/burst; `iostat`; instance bandwidth; snapshots/maintenance.
- **Likely root causes:** Burst exhausted, throughput ceiling, large queue, instance cap.
- **Safe fix:** Reduce load/provision correct performance. **Prevention:** baseline and headroom alarms.

### TS21 — High EC2 CPU but traffic unchanged

- **Symptom:** CPU saturation. **Likely layers:** process, release, runtime, host/dependency.
- **Check order and tools / signals:** Per-process/thread profile; deployment timeline; GC; system/steal/iowait; request traces.
- **Likely root causes:** Loop, GC pressure, crypto, retry storm, noisy process.
- **Safe fix:** Rollback/scale and cap harmful work. **Prevention:** profiling and release canaries.

### TS22 — Auto Scaling does not scale out

- **Symptom:** Alarm fires; desired capacity unchanged. **Likely layers:** metric/alarm, policy, limits, suspension.
- **Check order and tools / signals:** Alarm dimensions/state; scaling activity; min/max; cooldown/warmup; suspended processes; quotas.
- **Likely root causes:** Wrong metric, max reached, action missing, capacity failure.
- **Safe fix:** Correct policy/capacity limit with downstream safety. **Prevention:** scaling game days.

### TS23 — RDS connections exhausted

- **Symptom:** New connections rejected/time out. **Likely layers:** app pools/retries, DB limits, query duration.
- **Check order and tools / signals:** Connections by client/state; pool metrics; DB waits/top SQL; deploy event; proxy metrics.
- **Likely root causes:** Connection leak/storm, slow queries holding sessions.
- **Safe fix:** Limit clients, rollback, terminate only proven offenders. **Prevention:** pool budgets and load tests.

### TS24 — RDS read replica lag grows

- **Symptom:** Stale reads/recovery risk. **Likely layers:** writer workload, replica capacity, long transactions, network.
- **Check order and tools / signals:** Replica lag; write volume; replica CPU/IO; engine replication status; blocking/DDL.
- **Likely root causes:** Write burst, undersized replica, long transaction, heavy replica query.
- **Safe fix:** Reduce pressure/scale replica/remove blocking query. **Prevention:** lag SLO and routing policy.

### TS25 — RDS failover happened but app remains down

- **Symptom:** Database recovered; clients do not. **Likely layers:** DNS cache, pool/retry, transaction/application.
- **Check order and tools / signals:** RDS events/endpoint; DNS from app; connection pool; errors/traces; security path.
- **Likely root causes:** Stale DNS, infinite connection lifetime, retry storm.
- **Safe fix:** Recycle affected pools with bounded recovery. **Prevention:** regular failover testing.

## Containers and delivery

### TS26 — EKS pod is Pending

- **Symptom:** No node assignment. **Likely layers:** scheduler, capacity, constraints, PVC/IP.
- **Check order and tools / signals:** `describe pod` events; requests; taints/selectors/affinity; autoscaler; subnet IPs; PVC.
- **Likely root causes:** Insufficient resource, incompatible constraint, no IP, unbound volume.
- **Safe fix:** Add compatible capacity or correct requirement. **Prevention:** scheduling/capacity alerts.

### TS27 — EKS PVC is Pending

- **Symptom:** Claim never binds. **Likely layers:** StorageClass, CSI, IAM, topology, quota.
- **Check order and tools / signals:** PVC/events; StorageClass/binding mode; CSI pods/logs; pod scheduling; cloud API/KMS.
- **Likely root causes:** Missing provisioner, AZ conflict, CSI permission, quota.
- **Safe fix:** Repair CSI/compatible topology; preserve data objects. **Prevention:** storage integration tests.

### TS28 — Pod CrashLoopBackOff

- **Symptom:** Repeated restarts. **Likely layers:** process/config, resource, probes, dependency.
- **Check order and tools / signals:** `describe`; current/previous logs; exit code; OOM; config/secret; command/probes.
- **Likely root causes:** Bad configuration, crash, OOM, liveness too aggressive.
- **Safe fix:** Roll back or correct smallest cause. **Prevention:** startup probes and predeploy tests.

### TS29 — EKS service is unreachable

- **Symptom:** Pod works locally; Service/Ingress fails. **Likely layers:** endpoints, ports, policy, controller/LB.
- **Check order and tools / signals:** Service selectors/endpoints; targetPort; readiness; DNS; network policy; controller/LB logs.
- **Likely root causes:** Selector mismatch, no ready endpoints, wrong port, SG rule.
- **Safe fix:** Correct mapping/policy. **Prevention:** synthetic path tests.

### TS30 — EKS DNS failures

- **Symptom:** Pods intermittently cannot resolve. **Likely layers:** app cache, CoreDNS, service, node network/upstream.
- **Check order and tools / signals:** Query from pod; CoreDNS health/logs/metrics; endpoints; CPU/throttle; network policy; upstream.
- **Likely root causes:** CoreDNS saturation, blocked UDP/TCP 53, bad forwarding loop.
- **Safe fix:** Scale/restore path or correct rule. **Prevention:** DNS load/alarm tests.

### TS31 — EKS upgrade stalls

- **Symptom:** Nodes cannot drain or workloads fail. **Likely layers:** deprecated APIs, PDB, capacity, add-ons.
- **Check order and tools / signals:** Upgrade insights/docs; drain events; PDBs; spare capacity; add-on/skew; admission logs.
- **Likely root causes:** Impossible PDB, deprecated API, incompatible add-on, no surge capacity.
- **Safe fix:** Pause wave; restore capacity; correct constraint. **Prevention:** preflight and nonprod rehearsal.

### TS32 — ECS task cannot pull image

- **Symptom:** Task stops before start. **Likely layers:** image, execution role, ECR network/DNS, KMS.
- **Check order and tools / signals:** Stopped reason/service events; image/tag/platform; role; ECR/S3 endpoints or NAT; logs.
- **Likely root causes:** Missing execution permission, nonexistent tag, no egress.
- **Safe fix:** Restore correct immutable digest/path. **Prevention:** launch test per environment.

### TS33 — ECS deployment never stabilizes

- **Symptom:** Tasks cycle; service steady-state timeout. **Likely layers:** health, app startup, capacity, config.
- **Check order and tools / signals:** Service events; stopped reason; target health; container logs; grace period; secret/config.
- **Likely root causes:** Bad health path, insufficient capacity, crash, slow startup.
- **Safe fix:** Roll back task revision. **Prevention:** circuit breaker and canary.

### TS34 — Terraform state lock is stuck

- **Symptom:** Plan/apply refuses lock. **Likely layers:** active run, backend, lock object, process.
- **Check order and tools / signals:** CI runs/change system; lock owner/operation/time; backend availability; audit logs.
- **Likely root causes:** Aborted process or genuine active apply.
- **Safe fix:** Force-unlock only after proving no writer exists. **Prevention:** concurrency controls and timeouts.

### TS35 — Terraform partial apply

- **Symptom:** Some resources changed, apply failed. **Likely layers:** cloud reality, state, config/provider.
- **Check order and tools / signals:** Freeze runs; preserve logs/state; inspect completed actions; fresh plan; cloud API.
- **Likely root causes:** API error, permission/quota, dependency timing.
- **Safe fix:** Correct cause and apply reviewed reconciliation plan. **Prevention:** prechecks/smaller blast-radius states.

### TS36 — Terraform proposes unexpected destruction

- **Symptom:** Plan replaces/deletes critical resource. **Likely layers:** address/state, immutable field, module/provider change.
- **Check order and tools / signals:** Plan details; state list/show; config diff; provider changelog; moved/import blocks.
- **Likely root causes:** Rename without moved block, `for_each` key change, force-new attribute.
- **Safe fix:** Stop; repair address/migration and re-plan. **Prevention:** plan gates and module upgrade tests.

### TS37 — GitHub Actions OIDC fails

- **Symptom:** Cannot assume AWS role. **Likely layers:** workflow permission, token claims, trust policy, environment.
- **Check order and tools / signals:** `id-token: write`; job context; role/audience; sanitized claim inspection; CloudTrail.
- **Likely root causes:** `sub` condition mismatch, wrong role/account, environment claim.
- **Safe fix:** Match narrow expected claims. **Prevention:** federation smoke workflow.

### TS38 — CI pipeline is green but deployment is broken

- **Symptom:** Jobs passed; users fail. **Likely layers:** test quality, artifact/config, rollout verification.
- **Check order and tools / signals:** Artifact digest; deploy metadata; runtime SLO/traces; environment config; smoke-test coverage.
- **Likely root causes:** Tested different artifact, shallow health check, hidden config drift.
- **Safe fix:** Roll back/shift traffic. **Prevention:** promote same digest and SLO gates.

### TS39 — Jenkins queue grows indefinitely

- **Symptom:** Builds wait. **Likely layers:** labels/executors, cloud agents, locks, controller.
- **Check order and tools / signals:** Queue reason; node labels/status; provisioning logs; quotas; locks; controller CPU/heap.
- **Likely root causes:** Label mismatch, agent provisioning failure, locked resource.
- **Safe fix:** Restore matching agents/remove verified stale lock. **Prevention:** capacity and queue alarms.

### TS40 — Secret appears in CI logs

- **Symptom:** Credential exposed. **Likely layers:** job output, artifact/cache, logs, credential scope.
- **Check order and tools / signals:** Stop/limit access; identify value and all copies; audit usage; workflow/shell trace.
- **Likely root causes:** Debug echo, unsafe interpolation, unmasked derived value.
- **Safe fix:** Revoke/rotate, purge according to platform procedure, investigate. **Prevention:** short-lived identity and secret scanning.

## Observability and resilience

### TS41 — CloudWatch alarm did not fire

- **Symptom:** Impact occurred without notification. **Likely layers:** metric/dimension, periods, missing data, action route.
- **Check order and tools / signals:** Metric graph; alarm history/config; dimensions/statistic; missing data; SNS/on-call delivery.
- **Likely root causes:** Wrong dimension, sparse metric treated as good, action disabled.
- **Safe fix:** Correct and test alarm path. **Prevention:** synthetic alarm exercises.

### TS42 — Multi-Region failover passes health but users fail

- **Symptom:** DNS shifted, service unusable. **Likely layers:** data, identity/secrets, dependencies, capacity, client cache.
- **Check order and tools / signals:** End-to-end synthetic; replication/RPO; regional config/secrets/certs; quotas; DNS answers; traces.
- **Likely root causes:** Data not promoted, missing regional dependency, cold capacity, stale clients.
- **Safe fix:** Follow runbook; fail back only if safer and data direction is clear. **Prevention:** timed full-stack game days.

### TS43 — EC2 cannot reach S3

- **Symptom:** S3 API from EC2 times out or fails. **Likely layers:** DNS, NAT/gateway endpoint route, SG/NACL, endpoint/IAM/bucket/KMS policy.
- **Check order:** Identify timeout versus 403; resolve regional endpoint; inspect gateway endpoint association or NAT route; check flow; then authorization.
- **Tools/signals:** `curl`, AWS CLI debug/request ID, route table, Flow Logs, CloudTrail, policy simulator.
- **Likely root causes:** Endpoint not associated to subnet route table, endpoint-policy deny, missing object ARN or KMS decrypt.
- **Safe fix:** Correct only the route/policy tuple. **Prevention:** Runtime-identity connectivity test and IaC route/policy validation.

### TS44 — EKS HPA does not scale

- **Symptom:** Load rises but desired replicas stay unchanged. **Likely layers:** metric source, HPA config, resource requests, stabilization, scheduling.
- **Check order:** Inspect HPA conditions/current target; metric API/adapter; pod requests; min/max/policies; then Pending replicas/node capacity.
- **Tools/signals:** `kubectl describe hpa`, metrics API, pod/resource metrics, events, autoscaler logs.
- **Likely root causes:** Missing metrics, CPU request absent, wrong selector/metric, max reached, new pods unschedulable.
- **Safe fix:** Correct metric/request/config and load-test. **Prevention:** Exercise the complete HPA-to-node-scaling loop with alarms.

### TS45 — CPU is normal but latency is high

- **Symptom:** p95/p99 rises while average CPU looks healthy. **Likely layers:** queue, lock, storage, network, dependency, pool, hot shard.
- **Check order:** Trace critical path; compare tail versus average; inspect concurrency/queue/pools; DB waits/storage; downstream and affected slice.
- **Tools/signals:** Distributed traces, RED metrics, thread/connection pools, DB waits, EBS/network metrics, deployment timeline.
- **Likely root causes:** Lock contention, exhausted pool, slow dependency, IO queue, retry amplification or uneven traffic.
- **Safe fix:** Shed/limit traffic, roll back or relieve proven bottleneck. **Prevention:** Tail-latency SLOs, capacity tests and dependency budgets.

### TS46 — EC2 disk is full

- **Symptom:** Writes fail, application crashes or instance becomes unhealthy. **Likely layers:** filesystem bytes/inodes, logs, deleted-open files, application retention.
- **Check order:** Check space and inodes; locate growth; inspect open deleted files/mounts; stop nonessential writer; validate expansion path.
- **Tools/signals:** `df -h`, `df -i`, `du`, `lsof`, log/service metrics and EBS configuration.
- **Likely root causes:** Unbounded logs/temp, inode exhaustion, wrong mount, database/log retention.
- **Safe fix:** Remove/rotate only owned recoverable data or expand safely. **Prevention:** Retention, forecasting and tested cleanup/expansion runbook.

### TS47 — RDS connection timeout

- **Symptom:** Application cannot open DB connection. **Likely layers:** DNS, route, SG/NACL, DB state, connections/TLS/auth.
- **Check order:** Resolve endpoint; test TCP from application path; inspect SG source and route/NACL; DB events/connections; then TLS/credentials.
- **Tools/signals:** DNS/TCP probe, Reachability Analyzer, Flow Logs, RDS events/metrics, DB/auth logs.
- **Likely root causes:** Wrong SG reference, stale endpoint, no hybrid return route, max connections, failover/retry issue.
- **Safe fix:** Correct narrow network/capacity/client cause. **Prevention:** Synthetic connection checks, pool budgets and failover rehearsals.

## Interview closeout

After any scenario, state: impact and timeline; evidence; stabilization; root cause; corrective action; verification; monitoring/runbook/test that prevents recurrence. Do not claim you operated an incident you only studied—explain the method and experience boundary.
