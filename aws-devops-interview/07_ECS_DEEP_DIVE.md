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

## Identity boundary

The **execution role** lets the ECS/Fargate agent pull images, publish logs, and retrieve referenced configuration. The **task role** is used by application code calling AWS APIs. Scope each task role; do not hide application permissions in the host role.

## Deployment

Tune minimum/maximum healthy percentages, health-check grace, circuit breaker/rollback, alarms, capacity, and load-balancer deregistration. Blue/green enables controlled traffic shift and fast rollback but needs duplicate capacity and careful database compatibility.

## Task will not start

Check service events and stopped reason, capacity/subnet IPs, platform/CPU architecture, ECR image/tag, endpoint/NAT/DNS path, execution-role permissions, secrets/KMS, log configuration, port conflicts, and resource reservations.

## Running but unhealthy

Compare container, ECS, and target-group health. Validate container port mapping, listener/target type, security groups, health path/status, start period, bind address, dependency startup, and application logs. A container can be “running” while the service is unusable.

## Scaling

Service Auto Scaling changes task count; capacity providers change underlying capacity. Scale on a metric representing demand per unit—request count, queue depth, or utilization—while accounting for startup time and downstream limits.

## ECS versus EKS

Choose ECS for simpler AWS-native orchestration and smaller operational surface. Choose EKS for Kubernetes APIs/ecosystem, portability requirements, or organizational platform standards. Neither choice fixes poor workload boundaries, unsafe releases, or weak telemetry.

## Official reference

- [IAM roles for ECS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-iam-role-overview.html)
