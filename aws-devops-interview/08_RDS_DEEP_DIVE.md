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

## Architecture

```text
application -> proxy/pool -> writer endpoint -> primary
                    \-----> reader endpoint -> replicas
backup/PITR + cross-Region recovery plan
```

## Performance method

Start at transaction latency and throughput, then database waits, top SQL, locks, connection count, buffer/cache, CPU/memory, storage latency/queue, replication lag, and downstream calls. High CPU may be legitimate workload, missing indexes, plan regression, excessive connections, or background maintenance.

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

## Slow query flow

1. Define query/fingerprint, onset, impact, and baseline.
2. Inspect database waits and execution plan—not only CPU.
3. Check locks, rows scanned, index/selectivity, statistics, temp spill, storage.
4. Correlate application release, parameter change, data growth, failover.
5. Stabilize safely; test index/query/parameter changes on representative data.
6. Verify regression and add performance guardrails.

## Memory hook

**HA ≠ read scale ≠ backup.** Name which requirement each feature satisfies.

## Official references

- [RDS high availability](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.html)
- [RDS read replicas](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html)
