# 07 — ECS Deep Dive

## Model

```text
cluster -> service -> task definition revision -> tasks -> containers
                         |                   |
                 execution role         task role
```

- A task definition is a versioned specification; a task is a running copy.
- A service maintains desired count, deployments, discovery, and load-balancer integration.
- Fargate removes host management; EC2 capacity providers provide instance-level control and may improve economics for steady scale.

An ECS cluster is a logical scheduling boundary for services/tasks and capacity. A task definition revision declares images, CPU/memory, ports, volumes, environment/secrets, logging, health, runtime platform and roles. Standalone tasks fit finite jobs; a service maintains desired count and deployments. ECS Exec provides audited interactive command execution through Systems Manager channels when enabled and authorized—use for diagnosis, not routine configuration drift.

## Identity boundary

The **execution role** lets the ECS/Fargate agent pull images, publish logs, and retrieve referenced configuration. The **task role** is used by application code calling AWS APIs. Scope each task role; do not hide application permissions in the host role.

## Deployment

Tune minimum/maximum healthy percentages, health-check grace, circuit breaker/rollback, alarms, capacity, and load-balancer deregistration. Blue/green enables controlled traffic shift and fast rollback but needs duplicate capacity and careful database compatibility.

With `awsvpc`, each task receives an ENI/network identity and task security groups; it is required for Fargate. Plan subnet IPs, endpoints/NAT, DNS and target-group target type. On EC2, bridge/host networking changes port sharing and isolation. Capacity providers associate services with Fargate, Fargate Spot or Auto Scaling capacity using base/weight strategies.

Service Auto Scaling changes desired tasks; the underlying EC2 capacity provider/ASG must also have room. Scale on request rate per target, queue backlog per task or another divisible demand signal. CloudWatch Logs commonly receives container output through a log driver; add structured request/version/task context and traces.

## Task will not start

Check service events and stopped reason, capacity/subnet IPs, platform/CPU architecture, ECR image/tag, endpoint/NAT/DNS path, execution-role permissions, secrets/KMS, log configuration, port conflicts, and resource reservations.

## Running but unhealthy

Compare container, ECS, and target-group health. Validate container port mapping, listener/target type, security groups, health path/status, start period, bind address, dependency startup, and application logs. A container can be “running” while the service is unusable.

## Scaling

Service Auto Scaling changes task count; capacity providers change underlying capacity. Scale on a metric representing demand per unit—request count, queue depth, or utilization—while accounting for startup time and downstream limits.

## ECS versus EKS

Choose ECS for simpler AWS-native orchestration and smaller operational surface. Choose EKS for Kubernetes APIs/ecosystem, portability requirements, or organizational platform standards. Neither choice fixes poor workload boundaries, unsafe releases, or weak telemetry.

### Fargate versus EC2

| Fargate | ECS on EC2 |
|---|---|
| No host patching/capacity bin-packing | Full instance/runtime/agent control |
| Per-task resource billing | Potentially efficient steady fleet/commitments |
| Stronger task-level infrastructure isolation | Containers share customer-managed host kernel |
| Constraints on privileged/host patterns | Supports specialized instances and host integrations |

## Ordered troubleshooting playbooks

### Task stops immediately

1. Read service events, stopped reason, container reason and exit code.
2. Validate task definition revision, CPU/memory/runtime architecture and command.
3. Check image pull, execution role, secret/KMS and log-driver initialization.
4. Inspect application logs/OOM/health and dependency startup.
5. Roll back to last healthy revision or correct the proven failure.

### Cannot pull image

1. Verify repository, tag/digest, platform architecture and Region.
2. Check task execution-role ECR permissions and repository policy.
3. Check DNS plus NAT or ECR API/DKR and S3 endpoint paths.
4. Inspect task stopped reason and CloudTrail/network signals.
5. Fix the exact reference/path/policy and launch a canary task.

### Task cannot reach RDS

1. Resolve RDS endpoint from task context; confirm port/engine.
2. Verify task ENI subnet routes and RDS subnet/network path.
3. Allow RDS ingress from the task SG; verify actual SG attachment.
4. Check NACL, DNS, database status/connections and TLS/authentication.
5. Distinguish timeout from credential/database rejection.

### ALB target unhealthy

1. Inspect target reason, task/container status and service events.
2. Verify target type/IP, container/host port, listener and process bind address.
3. Check health path/status/timeout/grace and ALB-to-task SG.
4. Correlate ALB access logs with application logs.
5. Correct health/port/app issue and verify before increasing rollout.

### Fargate networking issue

1. Confirm task ENI creation, subnet free IPs and SG.
2. Check routes, NAT/endpoints, NACL and DNS settings.
3. Validate required ECR, logs, secrets and application dependency paths.
4. Use Flow Logs/service stopped reasons to isolate rejection versus no route.
5. Correct zonal path and add launch/connectivity tests.

### Task-role permission issue

1. Capture caller identity from application and exact API error.
2. Ensure permission belongs on task role, not execution/host role.
3. Check task definition role ARN and newly deployed task revision.
4. Evaluate IAM/SCP/resource/endpoint/KMS policies.
5. Apply minimal permission and test through the application.

### Deployment stuck

1. Read service deployment and event timeline; compare desired/running/pending.
2. Check capacity, subnet IPs, placement constraints and image/config.
3. Check container and load-balancer health, grace/deregistration and min/max healthy settings.
4. Trigger circuit-breaker/known-good rollback if impact criteria are met.
5. Fix capacity/health cause and add pre-traffic/canary gates.

## Official reference

- [IAM roles for ECS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-iam-role-overview.html)
