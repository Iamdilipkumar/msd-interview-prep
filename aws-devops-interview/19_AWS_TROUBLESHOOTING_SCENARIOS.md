# 19 — AWS Troubleshooting Scenarios

## Universal incident loop

```text
scope/impact -> timeline/change -> layer hypothesis -> evidence -> stabilize
             -> smallest safe correction -> verify -> prevent
```

Preserve evidence, use read-only checks first, change one variable at a time, and name rollback criteria. The following 42 scenarios are deliberately compact; practice expanding each aloud.

## Network and edge

### TS01 — Private EC2 cannot reach the internet

- **Symptom:** Outbound HTTPS times out. **Layers:** DNS, subnet route, NAT, NACL/SG, host.
- **Check order/tools:** Resolve DNS; inspect source subnet route; NAT state/metrics; Flow Logs/Reachability Analyzer; `curl`; host proxy/firewall.
- **Likely root causes:** Route points to wrong/AZ-failed NAT, NAT absent, NACL blocks ephemeral return, DNS failure.
- **Safe fix:** Correct the explicit route/control; use service endpoint where appropriate. **Prevent:** Per-AZ egress design, alarms, route policy tests.

### TS02 — Public EC2 is unreachable

- **Symptom:** Connection timeout. **Layers:** addressing, IGW/route, SG, NACL, OS/service.
- **Check order/tools:** Confirm public address and route; SG source/port; NACL both ways; Flow Logs; status checks; listener/host firewall.
- **Likely root causes:** No public IP, wrong SG, process bound to localhost, return traffic blocked.
- **Safe fix:** Prefer Session Manager; narrowly correct access. **Prevent:** Avoid public admin ports, test IaC controls.

### TS03 — Inter-VPC traffic fails

- **Symptom:** Connected VPC workloads time out. **Layers:** CIDR, peering/TGW, routes, policies, DNS.
- **Check order/tools:** Check overlap and attachment state; forward/return routes; TGW route domains; SG/NACL; Flow Logs; Resolver.
- **Likely root causes:** Missing return route, non-transitive peering assumption, wrong TGW propagation.
- **Safe fix:** Add precise symmetric route. **Prevent:** Route-domain tests and IPAM.

### TS04 — Hybrid VPN is intermittent

- **Symptom:** Loss/reconnects to on-premises. **Layers:** tunnels/BGP, MTU, routes, firewall, application.
- **Check order/tools:** Tunnel/BGP metrics and logs; route preference; both tunnels; packet size/DF tests; device logs.
- **Likely root causes:** One tunnel unused/broken, asymmetric route, MTU, rekey, saturated internet path.
- **Safe fix:** Restore redundant routing or shift path. **Prevent:** Exercise both tunnels, Direct Connect plus VPN where justified.

### TS05 — NAT gateway connections fail under load

- **Symptom:** New outbound connections time out during peaks. **Layers:** application reuse, ports, NAT, destination.
- **Check order/tools:** NAT error/connection metrics; five-tuples/Flow Logs; client retry/pooling; destination limits.
- **Likely root causes:** Port exhaustion to one destination, connection leak, retry storm.
- **Safe fix:** Reuse connections, spread destinations/NAT IP capacity, rate limit. **Prevent:** Capacity alarms/load tests/endpoints.

### TS06 — VPC endpoint returns AccessDenied

- **Symptom:** Private connectivity works but API is denied. **Layers:** DNS/path, endpoint policy, IAM, resource policy, KMS.
- **Check order/tools:** Confirm endpoint DNS; CloudTrail principal/action/resource; simulate policies; inspect endpoint/bucket/key policy.
- **Likely root causes:** Endpoint-policy deny, wrong principal, missing KMS permission.
- **Safe fix:** Narrowly allow required tuple. **Prevent:** Contract tests from workload identity.

### TS07 — Route 53 change appears inconsistent

- **Symptom:** Some clients see old target. **Layers:** authoritative DNS, recursive/OS/app cache, connections.
- **Check order/tools:** `dig` via multiple resolvers; record/TTL/delegation; app cache; persistent connection state.
- **Likely root causes:** Cached TTL/negative response, wrong hosted zone, connection reuse.
- **Safe fix:** Validate new target and allow cache expiry/controlled traffic shift. **Prevent:** Lower TTL before planned change.

### TS08 — ALB returns 502

- **Symptom:** Bad Gateway. **Layers:** listener/target protocol, app process, connection/TLS.
- **Check order/tools:** ALB access logs/metrics; target health; app logs; direct target request; protocol/port/timeouts.
- **Likely root causes:** Target reset/malformed response, TLS mismatch, app crash.
- **Safe fix:** Roll back bad version or correct protocol. **Prevent:** contract/smoke tests and canary alarms.

### TS09 — ALB returns 503

- **Symptom:** Service unavailable. **Layers:** target registration/health, capacity, routing rule.
- **Check order/tools:** HealthyHostCount; service/ASG events; target group/AZ; rules; SG and health path.
- **Likely root causes:** No healthy targets, failed deployment, exhausted capacity.
- **Safe fix:** Restore last known-good targets/capacity. **Prevent:** minimum healthy gates and pre-traffic checks.

### TS10 — ALB returns 504

- **Symptom:** Gateway timeout. **Layers:** target, dependency, database, timeout chain.
- **Check order/tools:** TargetResponseTime/access logs; traces; app thread/connection pools; DB waits; downstream latency.
- **Likely root causes:** Slow query/dependency, pool starvation, mismatched timeouts.
- **Safe fix:** Shed load/rollback/scale bottleneck. **Prevent:** timeout budgets, load tests, bulkheads.

### TS11 — CloudFront serves stale content

- **Symptom:** Old asset despite origin update. **Layers:** behavior, cache key, TTL, invalidation, origin.
- **Check order/tools:** Response/cache headers; behavior order; key inputs; invalidation; origin version.
- **Likely root causes:** Reused object key with long TTL, wrong behavior, cached error.
- **Safe fix:** Publish versioned key or targeted invalidation. **Prevent:** content-hashed assets.

### TS12 — CloudFront/WAF returns 403

- **Symptom:** Viewer denied. **Layers:** WAF, distribution, signed access, origin policy.
- **Check order/tools:** WAF/CloudFront logs and request ID; rule match; hostname/cert; OAC bucket policy; origin response.
- **Likely root causes:** WAF false positive, invalid signature, origin access denial.
- **Safe fix:** Scoped WAF exclusion or policy repair. **Prevent:** count-mode rollout and synthetic tests.

## Identity, data, and compute

### TS13 — AssumeRole is denied

- **Symptom:** STS AccessDenied. **Layers:** caller permission, role trust, SCP/boundary/session, conditions.
- **Check order/tools:** Caller identity; CloudTrail; trust principal/conditions; caller `sts:AssumeRole`; policy simulator.
- **Likely root causes:** Wrong session principal, subject/external ID mismatch, SCP deny.
- **Safe fix:** Correct exact trust/caller rule. **Prevent:** automated federation tests.

### TS14 — `iam:PassRole` failure

- **Symptom:** Resource creation cannot pass execution role. **Layers:** deploy identity and target service.
- **Check order/tools:** CloudTrail action/resource; deploy policy; role ARN; `iam:PassedToService` condition.
- **Likely root causes:** Missing PassRole or role outside allowed path.
- **Safe fix:** Allow only required role/service. **Prevent:** standardized deploy-role patterns.

### TS15 — KMS decrypt denied

- **Symptom:** Encrypted object/secret cannot be read. **Layers:** IAM, key policy/grant, region/key state, context.
- **Check order/tools:** Exact principal/key ARN; CloudTrail; key policy/grants; SCP/endpoint; encryption context.
- **Likely root causes:** Key policy excludes role, wrong Region, context mismatch, disabled key.
- **Safe fix:** Narrow policy/grant correction. **Prevent:** integration tests and key-owner runbook.

### TS16 — S3 GetObject denied but List works

- **Symptom:** Keys list; object read fails. **Layers:** object ARN policy, bucket policy, ownership, KMS.
- **Check order/tools:** Caller/action/key; CloudTrail data event; object policy/ownership; KMS authorization.
- **Likely root causes:** Only bucket ARN allowed, explicit resource deny, missing decrypt.
- **Safe fix:** Correct object resource/KMS permission. **Prevent:** least-privilege tests for each API.

### TS17 — S3 upload is slow

- **Symptom:** Low throughput/high retries. **Layers:** client, path, request shape, S3/KMS.
- **Check order/tools:** Object size/concurrency; multipart use; latency/status/retries; NAT/endpoint; CPU/network; KMS throttle.
- **Likely root causes:** Serial upload, distant path, tiny requests, KMS throttling.
- **Safe fix:** Multipart/parallelism within limits or path correction. **Prevent:** representative transfer benchmarks.

### TS18 — EC2 launch fails

- **Symptom:** Pending then terminated or capacity error. **Layers:** quota/capacity, launch template, KMS/IAM, subnet.
- **Check order/tools:** ASG/EC2 reason; quotas; instance flexibility; subnet IPs; AMI/snapshot/KMS permissions.
- **Likely root causes:** AZ capacity, quota, invalid template, encrypted AMI key denial.
- **Safe fix:** Diversify types/AZs or correct template policy. **Prevent:** capacity strategy and launch canaries.

### TS19 — EC2 bootstrapping fails

- **Symptom:** Instance healthy but app absent. **Layers:** cloud-init, network, IAM, package/service.
- **Check order/tools:** Console/cloud-init logs; user-data exit; DNS/NAT; role; filesystem; systemd logs.
- **Likely root causes:** Non-idempotent script, repo unreachable, secret denied.
- **Safe fix:** Correct image/script and replace instance. **Prevent:** bake/test immutable AMIs.

### TS20 — EBS latency spikes

- **Symptom:** Application stalls with storage queue. **Layers:** workload IO, volume, instance, filesystem.
- **Check order/tools:** EBS latency/queue/IOPS/throughput/burst; `iostat`; instance bandwidth; snapshots/maintenance.
- **Likely root causes:** Burst exhausted, throughput ceiling, large queue, instance cap.
- **Safe fix:** Reduce load/provision correct performance. **Prevent:** baseline and headroom alarms.

### TS21 — High EC2 CPU but traffic unchanged

- **Symptom:** CPU saturation. **Layers:** process, release, runtime, host/dependency.
- **Check order/tools:** Per-process/thread profile; deployment timeline; GC; system/steal/iowait; request traces.
- **Likely root causes:** Loop, GC pressure, crypto, retry storm, noisy process.
- **Safe fix:** Rollback/scale and cap harmful work. **Prevent:** profiling and release canaries.

### TS22 — Auto Scaling does not scale out

- **Symptom:** Alarm fires; desired capacity unchanged. **Layers:** metric/alarm, policy, limits, suspension.
- **Check order/tools:** Alarm dimensions/state; scaling activity; min/max; cooldown/warmup; suspended processes; quotas.
- **Likely root causes:** Wrong metric, max reached, action missing, capacity failure.
- **Safe fix:** Correct policy/capacity limit with downstream safety. **Prevent:** scaling game days.

### TS23 — RDS connections exhausted

- **Symptom:** New connections rejected/time out. **Layers:** app pools/retries, DB limits, query duration.
- **Check order/tools:** Connections by client/state; pool metrics; DB waits/top SQL; deploy event; proxy metrics.
- **Likely root causes:** Connection leak/storm, slow queries holding sessions.
- **Safe fix:** Limit clients, rollback, terminate only proven offenders. **Prevent:** pool budgets and load tests.

### TS24 — RDS read replica lag grows

- **Symptom:** Stale reads/recovery risk. **Layers:** writer workload, replica capacity, long transactions, network.
- **Check order/tools:** Replica lag; write volume; replica CPU/IO; engine replication status; blocking/DDL.
- **Likely root causes:** Write burst, undersized replica, long transaction, heavy replica query.
- **Safe fix:** Reduce pressure/scale replica/remove blocking query. **Prevent:** lag SLO and routing policy.

### TS25 — RDS failover happened but app remains down

- **Symptom:** Database recovered; clients do not. **Layers:** DNS cache, pool/retry, transaction/application.
- **Check order/tools:** RDS events/endpoint; DNS from app; connection pool; errors/traces; security path.
- **Likely root causes:** Stale DNS, infinite connection lifetime, retry storm.
- **Safe fix:** Recycle affected pools with bounded recovery. **Prevent:** regular failover testing.

## Containers and delivery

### TS26 — EKS pod is Pending

- **Symptom:** No node assignment. **Layers:** scheduler, capacity, constraints, PVC/IP.
- **Check order/tools:** `describe pod` events; requests; taints/selectors/affinity; autoscaler; subnet IPs; PVC.
- **Likely root causes:** Insufficient resource, incompatible constraint, no IP, unbound volume.
- **Safe fix:** Add compatible capacity or correct requirement. **Prevent:** scheduling/capacity alerts.

### TS27 — EKS PVC is Pending

- **Symptom:** Claim never binds. **Layers:** StorageClass, CSI, IAM, topology, quota.
- **Check order/tools:** PVC/events; StorageClass/binding mode; CSI pods/logs; pod scheduling; cloud API/KMS.
- **Likely root causes:** Missing provisioner, AZ conflict, CSI permission, quota.
- **Safe fix:** Repair CSI/compatible topology; preserve data objects. **Prevent:** storage integration tests.

### TS28 — Pod CrashLoopBackOff

- **Symptom:** Repeated restarts. **Layers:** process/config, resource, probes, dependency.
- **Check order/tools:** `describe`; current/previous logs; exit code; OOM; config/secret; command/probes.
- **Likely root causes:** Bad configuration, crash, OOM, liveness too aggressive.
- **Safe fix:** Roll back or correct smallest cause. **Prevent:** startup probes and predeploy tests.

### TS29 — EKS service is unreachable

- **Symptom:** Pod works locally; Service/Ingress fails. **Layers:** endpoints, ports, policy, controller/LB.
- **Check order/tools:** Service selectors/endpoints; targetPort; readiness; DNS; network policy; controller/LB logs.
- **Likely root causes:** Selector mismatch, no ready endpoints, wrong port, SG rule.
- **Safe fix:** Correct mapping/policy. **Prevent:** synthetic path tests.

### TS30 — EKS DNS failures

- **Symptom:** Pods intermittently cannot resolve. **Layers:** app cache, CoreDNS, service, node network/upstream.
- **Check order/tools:** Query from pod; CoreDNS health/logs/metrics; endpoints; CPU/throttle; network policy; upstream.
- **Likely root causes:** CoreDNS saturation, blocked UDP/TCP 53, bad forwarding loop.
- **Safe fix:** Scale/restore path or correct rule. **Prevent:** DNS load/alarm tests.

### TS31 — EKS upgrade stalls

- **Symptom:** Nodes cannot drain or workloads fail. **Layers:** deprecated APIs, PDB, capacity, add-ons.
- **Check order/tools:** Upgrade insights/docs; drain events; PDBs; spare capacity; add-on/skew; admission logs.
- **Likely root causes:** Impossible PDB, deprecated API, incompatible add-on, no surge capacity.
- **Safe fix:** Pause wave; restore capacity; correct constraint. **Prevent:** preflight and nonprod rehearsal.

### TS32 — ECS task cannot pull image

- **Symptom:** Task stops before start. **Layers:** image, execution role, ECR network/DNS, KMS.
- **Check order/tools:** Stopped reason/service events; image/tag/platform; role; ECR/S3 endpoints or NAT; logs.
- **Likely root causes:** Missing execution permission, nonexistent tag, no egress.
- **Safe fix:** Restore correct immutable digest/path. **Prevent:** launch test per environment.

### TS33 — ECS deployment never stabilizes

- **Symptom:** Tasks cycle; service steady-state timeout. **Layers:** health, app startup, capacity, config.
- **Check order/tools:** Service events; stopped reason; target health; container logs; grace period; secret/config.
- **Likely root causes:** Bad health path, insufficient capacity, crash, slow startup.
- **Safe fix:** Roll back task revision. **Prevent:** circuit breaker and canary.

### TS34 — Terraform state lock is stuck

- **Symptom:** Plan/apply refuses lock. **Layers:** active run, backend, lock object, process.
- **Check order/tools:** CI runs/change system; lock owner/operation/time; backend availability; audit logs.
- **Likely root causes:** Aborted process or genuine active apply.
- **Safe fix:** Force-unlock only after proving no writer exists. **Prevent:** concurrency controls and timeouts.

### TS35 — Terraform partial apply

- **Symptom:** Some resources changed, apply failed. **Layers:** cloud reality, state, config/provider.
- **Check order/tools:** Freeze runs; preserve logs/state; inspect completed actions; fresh plan; cloud API.
- **Likely root causes:** API error, permission/quota, dependency timing.
- **Safe fix:** Correct cause and apply reviewed reconciliation plan. **Prevent:** prechecks/smaller blast-radius states.

### TS36 — Terraform proposes unexpected destruction

- **Symptom:** Plan replaces/deletes critical resource. **Layers:** address/state, immutable field, module/provider change.
- **Check order/tools:** Plan details; state list/show; config diff; provider changelog; moved/import blocks.
- **Likely root causes:** Rename without moved block, `for_each` key change, force-new attribute.
- **Safe fix:** Stop; repair address/migration and re-plan. **Prevent:** plan gates and module upgrade tests.

### TS37 — GitHub Actions OIDC fails

- **Symptom:** Cannot assume AWS role. **Layers:** workflow permission, token claims, trust policy, environment.
- **Check order/tools:** `id-token: write`; job context; role/audience; sanitized claim inspection; CloudTrail.
- **Likely root causes:** `sub` condition mismatch, wrong role/account, environment claim.
- **Safe fix:** Match narrow expected claims. **Prevent:** federation smoke workflow.

### TS38 — CI pipeline is green but deployment is broken

- **Symptom:** Jobs passed; users fail. **Layers:** test quality, artifact/config, rollout verification.
- **Check order/tools:** Artifact digest; deploy metadata; runtime SLO/traces; environment config; smoke-test coverage.
- **Likely root causes:** Tested different artifact, shallow health check, hidden config drift.
- **Safe fix:** Roll back/shift traffic. **Prevent:** promote same digest and SLO gates.

### TS39 — Jenkins queue grows indefinitely

- **Symptom:** Builds wait. **Layers:** labels/executors, cloud agents, locks, controller.
- **Check order/tools:** Queue reason; node labels/status; provisioning logs; quotas; locks; controller CPU/heap.
- **Likely root causes:** Label mismatch, agent provisioning failure, locked resource.
- **Safe fix:** Restore matching agents/remove verified stale lock. **Prevent:** capacity and queue alarms.

### TS40 — Secret appears in CI logs

- **Symptom:** Credential exposed. **Layers:** job output, artifact/cache, logs, credential scope.
- **Check order/tools:** Stop/limit access; identify value and all copies; audit usage; workflow/shell trace.
- **Likely root causes:** Debug echo, unsafe interpolation, unmasked derived value.
- **Safe fix:** Revoke/rotate, purge according to platform procedure, investigate. **Prevent:** short-lived identity and secret scanning.

## Observability and resilience

### TS41 — CloudWatch alarm did not fire

- **Symptom:** Impact occurred without notification. **Layers:** metric/dimension, periods, missing data, action route.
- **Check order/tools:** Metric graph; alarm history/config; dimensions/statistic; missing data; SNS/on-call delivery.
- **Likely root causes:** Wrong dimension, sparse metric treated as good, action disabled.
- **Safe fix:** Correct and test alarm path. **Prevent:** synthetic alarm exercises.

### TS42 — Multi-Region failover passes health but users fail

- **Symptom:** DNS shifted, service unusable. **Layers:** data, identity/secrets, dependencies, capacity, client cache.
- **Check order/tools:** End-to-end synthetic; replication/RPO; regional config/secrets/certs; quotas; DNS answers; traces.
- **Likely root causes:** Data not promoted, missing regional dependency, cold capacity, stale clients.
- **Safe fix:** Follow runbook; fail back only if safer and data direction is clear. **Prevent:** timed full-stack game days.

## Interview closeout

After any scenario, state: impact and timeline; evidence; stabilization; root cause; corrective action; verification; monitoring/runbook/test that prevents recurrence. Do not claim you operated an incident you only studied—explain the method and experience boundary.
