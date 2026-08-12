# 06 — EKS Deep Dive

## Architecture

```text
AWS-managed control plane
 API server / etcd / schedulers
          |
   cluster endpoint + IAM auth
          |
nodes/Fargate -> kubelet -> pods -> services/Ingress
     |             |          |
   VPC CNI       CSI/PVC    ALB/NLB
```

EKS manages the Kubernetes control plane; customers still own workload design, node/Fargate capacity, add-ons, Kubernetes authorization, upgrades, network policy, images, data, and observability.

### Component responsibilities

| Component | Role | Failure signal |
|---|---|---|
| API server | Kubernetes API/authentication/admission entrypoint | API latency/errors, client timeouts, audit logs |
| Scheduler | Assigns unscheduled pods to compatible nodes | Pod `Pending` events |
| Controllers | Reconcile desired and actual resources | Object status/events, controller logs |
| kubelet | Node agent starts/monitors pods and reports status | Node conditions, kubelet logs |
| kube-proxy/data plane | Implements Service forwarding according to mode | Service path fails although endpoints exist |
| CoreDNS | Cluster service discovery/forwarding | DNS errors, latency, saturation |
| VPC CNI | Pod IP/ENI lifecycle and network integration | IP allocation errors, CNI logs/metrics |

Managed node groups automate EC2 node lifecycle integration but workloads still need disruption-safe upgrades. Fargate schedules eligible pods without customer nodes but has workload/DaemonSet/storage/networking constraints. A namespace is an API/authorization/policy organization boundary, not a hard multitenancy boundary by itself.

## Networking

- Amazon VPC CNI normally assigns VPC addresses to pods. Plan subnet IP capacity, ENIs, instance limits, prefix delegation, and scale behavior.
- A Service provides stable discovery/load balancing. Ingress/Gateway resources require a controller; declarations alone do nothing.
- Security groups, NACLs, network policies, CoreDNS, kube-proxy/data plane, target registration, and application listeners are separate layers.

The AWS Load Balancer Controller reconciles Kubernetes Ingress/Service resources into ALB/NLB resources. Debug controller events/logs, IAM, subnet discovery/tags, annotations, listener rules, target type, SG and pod readiness—an Ingress object alone does not create a functioning path.

## Identity

| Mechanism | Purpose | Watch |
|---|---|---|
| Cluster access entries/IAM auth | Human/automation access to API | Separate AWS authentication from Kubernetes authorization |
| EKS Pod Identity | Role per service account via EKS Auth API and node agent | Supported SDK/add-on and agent requirements |
| IRSA | Web-identity role per service account using cluster OIDC issuer | Trust policy/OIDC/audience/subject correctness |
| Node role | Kubelet and node operations | Do not let pods inherit broad node permissions |

## Scheduling flow

```text
Pending pod -> scheduler constraints -> available node resources
            -> autoscaler decision -> cloud capacity -> kubelet -> image -> readiness
```

Check requests, limits, affinity/anti-affinity, topology spread, taints/tolerations, selectors, quotas, PVC binding, pod density/IPs, autoscaler logs, instance capacity, and admission policies.

### Workload controllers

- **Deployment:** stateless replica rollout through ReplicaSets.
- **DaemonSet:** one eligible pod per node, commonly for node agents.
- **StatefulSet:** stable pod identity/ordered lifecycle and per-pod volume claims; it does not itself replicate application data.
- **HPA:** changes replicas from metrics. CPU utilization is relative to requested CPU; missing requests/metrics can prevent expected scaling.
- **Cluster Autoscaler:** changes capacity of known node groups for unschedulable pods.
- **Karpenter concept:** provisions suitable EC2 capacity directly from workload constraints and can consolidate/disrupt nodes according to policies. Confirm current APIs/behavior before production design.

### Persistent storage

```text
Pod -> PVC -> StorageClass -> CSI controller -> cloud volume/filesystem
                      | topology/binding |
Node <- CSI node plugin <- attach/mount <- PV
```

EBS CSI supplies AZ-scoped block volumes; topology and single-node access modes affect rescheduling. EFS CSI provides shared regional NFS semantics with different latency/throughput/security characteristics. Snapshots are crash-consistent unless the application quiesces; define application-consistent backup and restore tests.

ConfigMaps hold non-secret configuration; Kubernetes Secrets are base64-encoded API objects, not automatically a full secret-management solution. Restrict RBAC, encrypt at rest, prevent logs/env leakage and use Pod Identity/IRSA with a managed secret retrieval pattern where appropriate.

## PVC Pending

1. `kubectl describe pvc` and events: storage class, access mode, requested size.
2. Confirm StorageClass/provisioner/CSI controller health and IAM.
3. Check `volumeBindingMode`; `WaitForFirstConsumer` needs a schedulable pod.
4. Validate topology/AZ compatibility, quotas, subnet/network/KMS access.
5. Inspect CSI controller/node logs and cloud API events.
6. Correct the smallest cause; do not delete stateful resources blindly.

## Pod failure patterns

| State | First checks |
|---|---|
| Pending | Events, scheduling constraints, capacity, PVC, IPs |
| ImagePullBackOff | Image/tag, ECR network, execution identity, rate limits |
| CrashLoopBackOff | Previous logs, exit code, command, config/secret, probes, OOM |
| Running not Ready | Readiness endpoint, dependency, port, probe thresholds |
| OOMKilled | Working set, leak, request/limit, node pressure |
| Evicted | Disk/memory/PID pressure, ephemeral storage |

## Ordered troubleshooting playbooks

### Pod Pending

1. `kubectl describe pod` and read scheduler events.
2. Check requests versus allocatable resources and quotas.
3. Check selectors, affinity/topology, taints/tolerations and admission.
4. Check PVC binding, pod/ENI IP capacity and autoscaler decision.
5. Add compatible capacity or correct the smallest invalid constraint.

### CrashLoopBackOff

1. Inspect current and `--previous` logs, exit code and termination reason.
2. Check command/args, ConfigMap/Secret, mount and dependency reachability.
3. Check OOM, startup/liveness probes and permissions.
4. Compare image digest/release with last healthy version.
5. Roll back or correct the proven cause; do not merely increase restart delay.

### ImagePullBackOff

1. Read event: image not found, auth, DNS or timeout.
2. Verify exact immutable image reference and architecture.
3. Check ECR route/endpoints/NAT/DNS and node/Fargate pull identity.
4. Check repository policy/KMS and registry rate limits.
5. Fix reference/path/permission and restart rollout safely.

### OOMKilled

1. Confirm termination reason/exit and container versus node pressure.
2. Compare working set to request/limit and recent traffic/release.
3. Inspect heap/native leak, cache, concurrency and temporary allocations.
4. Stabilize by rollback/load control/right-sizing with available node capacity.
5. Profile and add load/memory regression tests.

### Node NotReady

1. Inspect node conditions/events: memory, disk, PID, network, kubelet heartbeat.
2. Check EC2/status, kubelet/container runtime, certificate/time and CNI.
3. Cordon; evaluate graceful drain and replacement against PDB/state.
4. Restore capacity before terminating when availability requires it.
5. Fix node image/bootstrap/capacity issue and verify replacement automation.

### Cluster DNS failure

1. Query a Service and external name from an affected and healthy pod.
2. Inspect CoreDNS Deployment/pods/endpoints/logs/metrics/config.
3. Check CPU throttling, replica spread, network policy and TCP/UDP 53.
4. Trace upstream VPC Resolver/forwarding and detect loops/timeout.
5. Scale/correct the failed layer and run DNS synthetic checks.

### Service not reachable

1. Check Service selector and EndpointSlices for ready pod IPs.
2. Verify port/targetPort and process bind/listener.
3. Test pod IP then Service DNS/ClusterIP from a debug pod.
4. Check network policy, kube data plane, SG/NACL and client DNS.
5. Correct mapping/readiness/policy and validate full path.

### Ingress returns 502/504

1. Check controller events/logs and ALB listener/rule/target health.
2. Verify Service endpoints, target type, ports, health path and pod readiness.
3. For 502 inspect protocol/TLS/reset; for 504 inspect target/dependency latency.
4. Correlate ALB access log, pod trace/log and network controls.
5. Roll back or repair the proven mapping/application bottleneck.

### Pod cannot access AWS

1. Capture actual caller identity and exact AccessDenied/request ID.
2. Verify service account and Pod Identity association/agent or IRSA annotation/OIDC trust.
3. Check SDK support/credential chain and ensure node credentials are not used.
4. Evaluate IAM, endpoint/resource and KMS policies.
5. Apply minimal permission/path correction and test from the workload.

### HPA not scaling

1. Inspect HPA conditions/current versus desired metrics.
2. Check Metrics Server/custom adapter and target metric availability.
3. Check resource requests because CPU percentage uses requests.
4. Check min/max, stabilization, scaling policies and unschedulable replicas.
5. Correct metric/config, then load-test the whole HPA-to-node-scaling loop.

### Nodes full

1. Compare requested/allocatable CPU, memory, pods, ephemeral storage and subnet IPs.
2. Find oversized requests, DaemonSet overhead and fragmentation/constraints.
3. Check autoscaler logs, limits, quotas and EC2 capacity.
4. Add diversified capacity or safely right-size/schedule workloads.
5. Forecast pod/IP capacity and enforce request standards.

### Failed managed node-group upgrade

1. Read node-group update error and EKS events.
2. Check surge capacity, EC2 quota/AZ capacity, subnet IPs and bootstrap/IAM.
3. Check PDBs, eviction blockers, local storage and termination grace.
4. Pause; restore healthy capacity and correct blocker before continuing.
5. Rehearse one-node/canary wave and monitor workload SLO.

### VPC CNI IP exhaustion

1. Check subnet available IPs, CNI logs/metrics and pod sandbox errors.
2. Count ENIs/pods/endpoints/LBs and instance ENI limits.
3. Add planned subnets/secondary CIDR or compatible node capacity.
4. Evaluate prefix delegation/CNI configuration against current guidance.
5. Alarm on subnet/IP headroom and include it in scaling forecasts.

## Probes and disruption

- Startup probe protects slow initialization from liveness restarts.
- Readiness controls traffic; liveness restarts a stuck container.
- PodDisruptionBudget limits voluntary disruption but does not create capacity or protect from all failures.
- Combine replicas across zones, topology spread/anti-affinity, graceful termination, `preStop`, realistic `terminationGracePeriodSeconds`, and load-balancer deregistration.

## Zero-downtime upgrade

1. Read EKS/Kubernetes version and add-on compatibility notes; identify deprecated APIs.
2. Back up manifests/data and prove rollback/recovery.
3. Upgrade nonproduction, then control plane one supported minor step at a time.
4. Upgrade CNI/CoreDNS/kube-proxy/CSI deliberately in tested order.
5. Create/upgrade node groups with surge capacity; cordon and drain respecting PDBs.
6. Canary workloads; validate SLOs, admission, DNS, storage, ingress, autoscaling.
7. Continue in waves. For hard rollback requirements, consider blue/green clusters and traffic shift.

“Zero downtime” depends on the application having multiple ready replicas, compatible APIs, spare capacity, disruption controls, and graceful connection behavior—not on the EKS upgrade button.

## Observability

Collect API audit/control-plane logs as required, node/kubelet/container logs, Kubernetes events, workload RED metrics, resource saturation, deployment changes, and traces. Correlate by cluster, namespace, workload, pod, version, node, AZ, and request ID; control label cardinality.

## Security baseline

Private or controlled API access, least-privilege access entries/RBAC, Pod Identity/IRSA, network policy, restricted pod security, image scanning/signing policy, secret encryption and external secret retrieval, patched nodes/add-ons, admission guardrails, audit logging, and isolated accounts/clusters according to blast radius.

## Official references

- [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)
- [EKS cluster upgrades](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html)
- [EKS best practices](https://docs.aws.amazon.com/eks/latest/best-practices/introduction.html)
