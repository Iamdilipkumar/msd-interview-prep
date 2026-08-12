# 08 — RDS Deep Dive

## Separate availability, scale, and recovery

| Capability | Primary purpose | Important qualifier |
|---|---|---|
| Multi-AZ deployment | Availability/failover | Not a read-scaling strategy for classic standby designs |
| Read replica | Read scale / DR option | Asynchronous; lag and promotion procedures matter |
| Automated backup/PITR | Restore to a time within retention | Restore creates a new DB; rehearse cutover |
| Snapshot | Durable point-in-time copy | Manual snapshots need lifecycle/governance |
| Aurora replicas | Read scale and failover targets | Connections/endpoints and replica lag still matter |
| Global Database | Cross-Region reads/DR | Cost, lag, write-forward strategy, promotion/failback |

RDS supports managed relational engines with service-specific version/feature differences. A **DB subnet group** supplies subnets across AZs for placement; security groups control network reachability. A **parameter group** configures engine settings, some dynamically and some after reboot. An **option group** enables supported engine-specific features. Treat both as versioned configuration and test changes.

Automated backups support PITR within retention; a manual snapshot persists until deleted. A restore creates a new DB and requires endpoint, subnet/security, parameter/option, secret, certificate and application cutover validation. Backup, snapshot and PITR are mechanisms; recovery requires a timed runbook.

## Architecture

```text
application -> proxy/pool -> writer endpoint -> primary
                    \-----> reader endpoint -> replicas
backup/PITR + cross-Region recovery plan
```

## Performance method

Start at transaction latency and throughput, then database waits, top SQL, locks, connection count, buffer/cache, CPU/memory, storage latency/queue, replication lag, and downstream calls. High CPU may be legitimate workload, missing indexes, plan regression, excessive connections, or background maintenance.

Performance Insights/current database observability capabilities expose database load and wait/top-SQL views for supported engines; CloudWatch exposes infrastructure metrics and events. Enhanced Monitoring adds OS-level visibility. Use engine-native logs/EXPLAIN carefully. Names and retention/features evolve, so confirm current service documentation.

Storage autoscaling can increase allocated storage when conditions are met; it is not instantaneous rescue, does not shrink storage, and cannot repair unbounded log/data growth. Plan maximum threshold, IOPS/throughput and free-space alarms. Vertical scaling changes instance capacity; horizontal read scaling adds replicas/caches/sharding/application complexity and does not automatically scale writes.

## Connection storms

Stabilize by limiting concurrency, scaling application carefully, using correctly sized pools/RDS Proxy where appropriate, and protecting the database. Diagnose deployment fan-out, retry loops, idle connections, max connection configuration, query duration, and failover behavior.

## Failover readiness

- Use DNS endpoints, not instance addresses; configure sane DNS caching.
- Retry transient connection failures with bounded exponential backoff and jitter.
- Make transactions/idempotency safe and failover-test clients.
- Monitor replication lag, connections, storage, waits, and transaction SLOs.
- Keep schema changes backward-compatible during rolling application releases.

## Encryption and access

Use private network placement, narrow security groups, TLS, IAM database authentication where appropriate, Secrets Manager rotation, encrypted storage/backups, and audit/database logs based on requirements. KMS key policy and grants affect snapshot copy/restore.

Maintenance windows schedule eligible maintenance but do not guarantee every change occurs only there. Plan engine upgrades with compatibility tests, parameter/extension checks, replicas, snapshot/recovery, blue/green deployment capabilities where supported, and application connection behavior. Minor/major upgrade and rollback options differ by engine/version.

## Slow query flow

1. Define query/fingerprint, onset, impact, and baseline.
2. Inspect database waits and execution plan—not only CPU.
3. Check locks, rows scanned, index/selectivity, statistics, temp spill, storage.
4. Correlate application release, parameter change, data growth, failover.
5. Stabilize safely; test index/query/parameter changes on representative data.
6. Verify regression and add performance guardrails.

## Ordered production troubleshooting

### CPU at 100%

1. Confirm duration, transactions/second, connections and recent release/maintenance.
2. Inspect database load/waits and top SQL, not CPU alone.
3. Check plans, rows scanned, missing indexes, locks, vacuum/maintenance and connection storm.
4. Stabilize with workload throttling, rollback/read routing or safe scale-up.
5. Fix query/schema/pool and load-test before right-sizing.

### High database connections / max connections

1. Count by application, user, state and age; inspect pool metrics.
2. Compare total fleet pool maximum to DB memory/connection capacity.
3. Find leaks, idle-in-transaction and slow queries holding sessions.
4. Limit/recycle offending clients or use RDS Proxy where appropriate.
5. Set global connection budgets, backoff and failover tests.

### Storage full

1. Measure free space/growth rate and time to exhaustion.
2. Identify table/index/log/temp/transaction retention growth.
3. Stop nonessential writers; avoid unsafe manual deletion.
4. Increase storage/maximum autoscaling threshold if supported and sufficient.
5. Correct retention/query/schema growth and alert forecasted exhaustion.

### Replica lag

1. Measure engine-specific lag and replication/apply state.
2. Check writer write rate/long transactions/DDL.
3. Check replica CPU, IO, locks and heavy read workload.
4. Scale/remove blocker and route consistency-sensitive reads away.
5. Define lag SLO, alarms and promotion safety criteria.

### Connection timeout

1. Resolve endpoint and confirm client has current DNS answer/port.
2. Check routes, DB subnet group placement, SG source, NACL and TLS.
3. Check DB status, connection limit, proxy/pool and authentication logs.
4. Test from the same application network/identity.
5. Correct network or capacity cause; never expose the DB publicly as a shortcut.

### Failover and DNS endpoint behavior

1. Confirm RDS event/failover target and writer endpoint resolution.
2. Measure client DNS cache, connection lifetime and retry behavior.
3. Check in-flight transaction errors/idempotency and pool refresh.
4. Restore user traffic with bounded backoff/reconnection; avoid retry storm.
5. Rehearse and measure full application RTO, not only DB promotion.

### IOPS bottleneck

1. Correlate DB waits/transaction latency with read/write latency, IOPS, throughput and queue.
2. Identify query IO pattern, cache hit, scans/temp spill and checkpoint/maintenance.
3. Check storage and instance bandwidth ceilings/burst behavior.
4. Optimize query/index/cache or provision correct storage/instance capacity.
5. Validate under peak load with headroom alarms.

## Fast comparisons

| Comparison | Strong answer |
|---|---|
| Multi-AZ vs read replica | Failover availability versus asynchronous read scale; exact engine/deployment behavior matters |
| Backup vs snapshot vs PITR | Managed retention/log-based restore versus retained point copy versus selecting a recoverable time |
| RDS vs Aurora | Managed standard engine deployments versus AWS-designed compatible cluster/storage architecture; benchmark and validate compatibility |
| Vertical vs horizontal | Bigger single unit is simpler but bounded; replicas/shards/caches add scale plus consistency/operations complexity |

## Memory hook

**HA ≠ read scale ≠ backup.** Name which requirement each feature satisfies.

## Official references

- [RDS high availability](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.html)
- [RDS read replicas](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html)
