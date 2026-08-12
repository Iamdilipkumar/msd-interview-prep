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

### AMIs, Nitro, placement, and lifecycle

- **AMI:** launch template for root volume, architecture, boot mode and permissions. Build reproducibly, patch/scan, test boot and deprecate old images.
- **Nitro concept:** AWS hardware/software offload architecture underlying many current instance families, enabling strong isolation and high network/EBS performance. Capabilities vary by type; verify rather than assume.
- **Instance store:** physically attached ephemeral block storage available only on selected instance types. Use for replicated cache/scratch data, never as the sole copy of important data.
- **Placement group:** cluster favors low-latency placement, spread separates a small number of critical instances, and partition separates groups of racks. Choose only for a workload requirement and understand launch/capacity constraints.
- **Key pair:** boot-time public key injection for supported OS access. Prefer Systems Manager Session Manager for controlled administration; a key pair does not grant IAM permission.
- **Termination protection:** guard against selected accidental termination calls; it is not backup and does not stop Auto Scaling replacement or every termination path.
- **Lifecycle:** pending -> running -> stopping/stopped (EBS-backed) -> shutting-down -> terminated. Instance-store data is ephemeral; public addressing and host placement can change.

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

Launch templates version the AMI, instance type, networking, storage, IAM and bootstrap settings used by Auto Scaling. Promote a tested version and roll with instance refresh/canary capacity. Auto Scaling replaces unhealthy instances; it does not understand application correctness unless health and alarms represent it.

## Ordered troubleshooting flows

### Instance unreachable / SSH failure

1. Separate instance status, network timeout, TCP refusal and authentication failure.
2. Check system/instance status, public/private path, routes, SG and NACL both ways.
3. Verify `sshd` listener, host firewall, disk/inodes and boot logs via SSM/serial/console methods.
4. Verify username, key permissions and authorized key—not IAM role.
5. Prefer repairing/replacing from a known image; avoid opening SSH broadly.

### CPU high

1. Confirm duration, affected instances, requests and deployment timeline.
2. Identify process/thread and user/system/steal/I/O-wait distribution.
3. Inspect GC, loops, retries, hot keys, crypto and downstream latency.
4. Stabilize with rollback, safe scale-out or rate limiting.
5. Profile/fix and add capacity/SLO tests rather than permanently over-sizing.

### Disk full

1. Check bytes and inodes by filesystem; find largest/newest growth safely.
2. Check deleted-open files, logs, container layers, temp files and database behavior.
3. Stop nonessential writers; rotate/archive only owned data.
4. Expand EBS/filesystem if appropriate and validate application recovery.
5. Add retention, inode/space forecasting and tested cleanup runbooks.

### Memory pressure

1. Confirm application impact, free/available memory, swap, OOM events and process working sets.
2. Correlate leak, cache, concurrency, release and container/cgroup limits.
3. Shed load/rollback/restart only with evidence and state-safety assessment.
4. Profile heap/native memory and right-size application/instance.
5. Alarm on sustained pressure and test peak workload.

### EBS latency

1. Correlate application latency with EBS latency, queue, IOPS and throughput.
2. Use `iostat` to inspect IO size, await/utilization and filesystem behavior.
3. Check volume burst/provisioned limits and instance EBS bandwidth ceiling.
4. Check snapshots, encryption/KMS dependencies, RAID/filesystem and noisy processes.
5. Reduce pressure or provision the correct volume/instance performance, then verify.

### Status checks fail

1. Determine system, instance or attached-EBS check and incident scope.
2. For system impairment, recover/replace according to architecture.
3. For instance impairment, inspect console/serial logs, kernel, network and disk.
4. Restore service with ASG replacement or controlled recovery; preserve evidence.
5. Fix image/configuration and validate automatic health replacement.

### Boot failure

1. Read console/system log and screenshot/serial output.
2. Check AMI architecture/boot mode, root mapping, filesystem and kernel/initramfs.
3. Check user data/cloud-init for blocking/reboot/failure and dependency access.
4. Attach root volume to a recovery instance only under controlled procedure, or replace.
5. Add image launch tests across intended instance families/AZs.

### Metadata credential issue

1. Confirm role/profile attachment and `aws sts get-caller-identity` result.
2. Check SDK credential precedence for stale environment/profile credentials.
3. Check IMDSv2 token flow, endpoint setting, hop limit and proxy/no-proxy.
4. Evaluate role/target policy after proving credentials are delivered.
5. Correct runtime configuration without weakening IMDS protections.

### Application inaccessible

1. Resolve DNS and test LB/listener or instance destination and port.
2. Check load-balancer target health, SG/NACL/routes and host listener/bind address.
3. Inspect process/service logs, configuration, certificate and dependencies.
4. Compare last known-good release and stabilize via rollback/healthy capacity.
5. Add end-to-end synthetic monitoring, not only instance CPU alarms.

### Spot interruption / capacity loss

1. Confirm interruption/rebalance event and affected desired capacity.
2. Drain/checkpoint work and stop new assignments.
3. Let ASG/capacity provider diversify across types/AZs or use On-Demand fallback.
4. Verify replacement health and queue/backlog recovery.
5. Design idempotent workloads and regularly test interruptions.

## Troubleshooting high CPU

1. Confirm timeframe, scope, traffic, and user impact.
2. Identify process/thread and CPU type: user, system, steal, I/O wait.
3. Correlate deployment, scaling, request rate, GC, dependency latency.
4. Stabilize with scale-out/rollback/rate limiting if safe.
5. Profile and correct code/query/configuration; load-test prevention.

## Memory hook

**Image, identity, interface, IO, initialization.** Those five “I”s explain most EC2 launches.
