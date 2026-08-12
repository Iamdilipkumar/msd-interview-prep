# 18 — Docker and Kubernetes DevOps

## Container image principles

- Multi-stage builds; minimal runtime image; pin base image by digest where assurance requires it.
- Run as non-root, drop capabilities, use read-only filesystem where possible.
- No secrets in layers, build arguments, history, or registry metadata.
- Generate SBOM, scan, sign/attest, and promote the same digest.
- Use `.dockerignore`; order layers for stable dependency caching.

An image is an immutable layered filesystem/config template; a container is a running process using that image plus writable runtime state. Containers share the host kernel, so namespaces/cgroups and runtime controls are isolation boundaries—not separate virtual machines. A registry stores manifests/layers by tag and digest; deploy by digest for immutability.

```dockerfile
FROM golang:1.24 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app ./cmd/app

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

The example demonstrates multi-stage build and non-root runtime; pin base digests and verify current language/runtime versions in a real implementation.

## Kubernetes workload contract

```text
Deployment -> ReplicaSet -> Pod -> containers
    | strategy     |          | requests/limits + probes + securityContext
Service -> endpoints/EndpointSlices -> Ready pods
Ingress/Gateway -> controller -> external load balancer
```

Kubernetes is a reconciliation system: users submit desired objects to the API, controllers converge them, the scheduler chooses nodes and kubelets run pods. ConfigMaps provide configuration; Secrets need RBAC/encryption/external lifecycle; Services select ready pod endpoints; controllers maintain replicas and rollout state.

## Resources and quality of service

Requests drive scheduling; limits constrain runtime. CPU throttles at its limit; memory overrun can lead to OOM termination. Set requests from observed demand and tune limits from risk, load tests, and node capacity. Avoid both unbounded workloads and arbitrary equal request/limit defaults.

## Safe rollout

Use multiple replicas, readiness/startup probes, controlled surge/unavailable settings, topology spread, PDBs for voluntary disruptions, graceful termination, and backward-compatible dependencies. Watch rollout status and service SLOs; pause/rollback on evidence.

For EKS, connect every Kubernetes object to AWS dependencies: VPC CNI supplies pod IPs, AWS Load Balancer Controller builds ALB/NLB paths, CSI drivers provision EBS/EFS, Pod Identity/IRSA supplies AWS permissions, and node groups/Fargate supply runtime capacity. A YAML-correct object can still fail because an AWS route, IAM/KMS policy, subnet IP pool or quota is wrong.

## Docker and Kubernetes interview comparisons

| Concept | Strong distinction |
|---|---|
| Image vs container | Immutable template versus running process |
| CMD vs ENTRYPOINT | Default arguments/command versus primary executable semantics; forms interact |
| Volume vs image layer | Persistent/runtime data versus immutable build content |
| Deployment vs StatefulSet | Interchangeable stateless replicas versus stable identity/storage ordering |
| Service vs Ingress | Stable service endpoint versus external HTTP routing declaration |
| ConfigMap vs Secret | Non-secret config versus sensitive API object; both need access governance |
| Request vs limit | Scheduling reservation versus runtime cap |
| Namespace vs cluster/account | Logical policy scope versus stronger blast-radius boundary |

## Debugging order

```text
desired state -> events -> pod state -> current/previous logs
-> process/listener -> Service endpoints -> DNS/network policy
-> ingress/LB -> dependencies -> node/cloud layer
```

Useful commands:

```bash
kubectl get deploy,rs,pods,svc,endpoints -n <namespace>
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --previous
kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp
kubectl rollout status deployment/<name> -n <namespace>
kubectl auth can-i <verb> <resource> --as=<identity>
```

## Supply-chain controls

Protect source and branches; pin dependencies; isolate builds; scan code/dependencies/images/IaC; sign artifacts; enforce allowed registries/signatures/admission policies; produce provenance; use short-lived deploy identity; and continuously scan running inventory. Scanning is a signal, not proof of exploitability or safety.

## Common traps

- “Containers are lightweight VMs.” They share the host kernel.
- “Liveness should test every dependency.” Shared dependency failure can trigger a restart storm.
- “Latest tag will update pods.” Tags are mutable and pull behavior varies; deploy immutable digests.
- “Rollback fixes everything.” Data/schema/external side effects may be irreversible.
