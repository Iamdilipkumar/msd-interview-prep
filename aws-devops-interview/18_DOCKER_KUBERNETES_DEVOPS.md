# 18 — Docker and Kubernetes DevOps

## Container image principles

- Multi-stage builds; minimal runtime image; pin base image by digest where assurance requires it.
- Run as non-root, drop capabilities, use read-only filesystem where possible.
- No secrets in layers, build arguments, history, or registry metadata.
- Generate SBOM, scan, sign/attest, and promote the same digest.
- Use `.dockerignore`; order layers for stable dependency caching.

## Kubernetes workload contract

```text
Deployment -> ReplicaSet -> Pod -> containers
    | strategy     |          | requests/limits + probes + securityContext
Service -> endpoints/EndpointSlices -> Ready pods
Ingress/Gateway -> controller -> external load balancer
```

## Resources and quality of service

Requests drive scheduling; limits constrain runtime. CPU throttles at its limit; memory overrun can lead to OOM termination. Set requests from observed demand and tune limits from risk, load tests, and node capacity. Avoid both unbounded workloads and arbitrary equal request/limit defaults.

## Safe rollout

Use multiple replicas, readiness/startup probes, controlled surge/unavailable settings, topology spread, PDBs for voluntary disruptions, graceful termination, and backward-compatible dependencies. Watch rollout status and service SLOs; pause/rollback on evidence.

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
