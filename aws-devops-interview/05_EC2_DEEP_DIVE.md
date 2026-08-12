# 05 — EC2 Deep Dive

## Lifecycle and dependencies

```text
AMI + instance type + subnet/ENI/SG + IAM role + EBS + user data
                         -> boot -> health -> register -> serve -> terminate
```

An EC2 problem can be control plane, placement/capacity, network, guest OS, storage, runtime, or dependency. Diagnose the layer before replacing the instance.

## Key choices

| Decision | Questions |
|---|---|
| Instance family | CPU, memory, accelerator, local storage, network/EBS bandwidth? |
| Purchase model | Baseline, interruptibility, duration, flexibility? |
| AMI | Patched, immutable, tested, traceable, minimal? |
| Storage | EBS durability/performance; instance-store ephemerality? |
| Network | ENI/IP capacity, placement, enhanced networking, path? |
| Identity | Narrow instance profile; IMDSv2; no static keys? |

Use On-Demand for flexibility, Savings Plans/Reservations for measured steady usage, and Spot for interruption-tolerant capacity with diversification and graceful termination handling.

## EBS performance

Latency, IOPS, throughput, IO size, queue depth, and instance EBS limits interact.

```text
throughput ~= IOPS x average IO size
effective ceiling = min(volume limit, instance limit, path/application limit)
```

High IOPS with a slow application can mean small inefficient IO, high latency/queueing, filesystem locks, CPU saturation, database contention, synchronous dependency calls, or exhausted instance bandwidth. Correlate volume metrics with OS tools (`iostat`, `vmstat`, filesystem space/inodes) and application traces.

## Bootstrap failures

Check console/system logs, instance status checks, cloud-init/user-data logs, package repository/DNS/NAT access, IAM permissions, secret retrieval, filesystem capacity, command idempotency, and service status. User data commonly runs only on first boot unless explicitly configured otherwise.

## Status checks

- **System status:** underlying AWS infrastructure path; consider recover/redeploy.
- **Instance status:** guest OS/network/configuration; inspect logs and use Systems Manager or serial console where available.
- **Attached EBS status:** storage reachability/I/O impairment signal.

## Safe operating model

- Build immutable versioned images and replace instances through an Auto Scaling group.
- Patch through controlled image pipelines or Systems Manager with rings and rollback.
- Keep state outside disposable compute where possible.
- Use Session Manager to reduce inbound administration paths.
- Alarm on service-level signals, not CPU alone.

## Troubleshooting high CPU

1. Confirm timeframe, scope, traffic, and user impact.
2. Identify process/thread and CPU type: user, system, steal, I/O wait.
3. Correlate deployment, scaling, request rate, GC, dependency latency.
4. Stabilize with scale-out/rollback/rate limiting if safe.
5. Profile and correct code/query/configuration; load-test prevention.

## Memory hook

**Image, identity, interface, IO, initialization.** Those five “I”s explain most EC2 launches.
