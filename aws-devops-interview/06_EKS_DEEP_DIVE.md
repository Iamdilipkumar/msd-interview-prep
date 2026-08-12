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

## Networking

- Amazon VPC CNI normally assigns VPC addresses to pods. Plan subnet IP capacity, ENIs, instance limits, prefix delegation, and scale behavior.
- A Service provides stable discovery/load balancing. Ingress/Gateway resources require a controller; declarations alone do nothing.
- Security groups, NACLs, network policies, CoreDNS, kube-proxy/data plane, target registration, and application listeners are separate layers.

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
