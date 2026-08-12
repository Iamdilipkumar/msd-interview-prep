# 15 — CI/CD Pipeline Design

## Goal

A pipeline should make the safe path repeatable: build once, prove quality, promote the same immutable artifact, constrain authority, detect failure, and recover quickly.

```text
commit -> checks -> build/SBOM/sign -> artifact registry
       -> deploy test -> integration/security -> approval
       -> canary/blue-green prod -> verify SLO -> promote or rollback
```

## Controls by stage

| Stage | Evidence/control |
|---|---|
| Source | Protected branches, review, signed/traceable changes |
| Build | Pinned dependencies, isolated runner, reproducible artifact |
| Test | Unit, contract, integration, performance/security proportional to risk |
| Artifact | Immutable digest, provenance, SBOM, scan/signature |
| Deploy | Short-lived environment role, change diff, concurrency control |
| Verify | Smoke tests, metrics/logs, automated rollback threshold |

## Deployment strategies

| Strategy | Benefit | Tradeoff |
|---|---|---|
| Rolling | Efficient capacity | Mixed versions; rollback takes another rollout |
| Blue/green | Fast traffic switch/rollback | Duplicate capacity and data/schema complexity |
| Canary | Limits initial blast radius | Requires good telemetry and representative traffic |
| Recreate | Simple | Downtime |

Database changes should use expand/contract: add backward-compatible schema, deploy code using both versions, migrate/validate data, switch reads/writes, then remove obsolete schema later. Application rollback cannot undo destructive schema changes safely.

## Failure handling

- Prevent overlapping production deployments.
- Keep build and deployment identities separate.
- Time out stuck jobs and make retryable steps idempotent.
- Define rollback versus roll-forward criteria before release.
- Preserve deployment metadata: commit, artifact digest, config, approver, timestamps.
- A rollback must be regularly tested, including configuration and data compatibility.

## Pipeline security

Use OIDC federation, least-privilege roles per environment, protected environments/approvals, masked secrets, trusted actions/plugins pinned to immutable versions, isolated ephemeral runners where possible, dependency/provenance controls, and restricted fork behavior. Treat pull requests as untrusted input.

## Senior answer

“I separate artifact creation from promotion. A reviewed commit produces one immutable, scanned, traceable artifact. Environment-specific deployment assumes short-lived least-privilege roles, applies policy and approval gates, releases through canary or blue/green, and evaluates service-level signals. Database changes remain backward-compatible. The system blocks concurrent releases and automatically stops or rolls back on predefined evidence.”
