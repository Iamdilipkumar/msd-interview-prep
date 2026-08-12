# 15 — CI/CD Pipeline Deep Dive

## Goal

A pipeline should make the safe path repeatable: build once, prove quality, promote the same immutable artifact, constrain authority, detect failure, and recover quickly.

```text
commit -> checks -> build/SBOM/sign -> artifact registry
       -> deploy test -> integration/security -> approval
       -> canary/blue-green prod -> verify SLO -> promote or rollback
```

**CI** continuously integrates small changes through automated checks and a reproducible build. **Continuous delivery** keeps a releasable artifact ready behind controlled promotion; **continuous deployment** automatically promotes every qualified change. The organization chooses the last step from risk and evidence—not terminology.

## From developer commit to production

```text
Developer branch
   |
   v
Pull request -> review + branch protection
   |
   +--> format/lint/unit tests
   +--> SAST + SCA + secret scan + IaC policy
   |
   v
Merge immutable commit
   |
   v
Reproducible build -> container image
   |                     |
   |                     +--> SBOM + vulnerability scan + signature/provenance
   v
Immutable registry digest (for example, ECR)
   |
   +--> deploy same digest to integration -> smoke + integration + DAST
   |
   +--> Terraform plan -> policy/cost/security review -> approval -> exact-plan apply
   |
   v
Production environment approval + short-lived OIDC role
   |
   v
Canary / blue-green / rolling deployment
   |
   +--> health + synthetic + SLO/error-budget validation
   |        |
   |        +--> fail: stop traffic, rollback or roll forward safely
   v
Promote traffic -> observe bake period -> record evidence
```

### Stage-by-stage explanation

1. **Commit/PR:** A short-lived branch opens a PR. Required reviews, code owners and protected checks enforce peer ownership. Trunk-based development favors small integrations; longer-lived release/GitFlow models may suit regulated/release trains but increase merge drift.
2. **Fast checks:** formatting, lint and unit tests fail quickly. Unit tests isolate code; integration tests prove boundaries with real/representative dependencies; contract tests protect service interfaces.
3. **Security checks:** SAST inspects source patterns, SCA evaluates dependencies/licenses, secret scanning finds credential-like material, IaC/container policy catches unsafe configuration. DAST exercises a running application. Findings need severity, exploitability, owner, SLA and expiring exception—not a blind “zero findings” slogan.
4. **Build:** A pinned, isolated toolchain creates a reproducible container/package. Multi-stage Docker builds minimize runtime content. Do not inject environment secrets into layers or build logs.
5. **Artifact:** Registry stores the immutable digest. Attach SBOM, scan result, signature/provenance and commit identity. SonarQube or equivalent quality analysis can enforce agreed coverage/duplication/security gates but does not replace review or tests.
6. **Infrastructure:** Terraform formatting/validation/tests/security checks precede a saved plan. Human/policy gate reviews destructive/replacement/network/IAM changes; apply uses that same plan under a one-writer lock.
7. **Promotion:** Environment-specific configuration and short-lived OIDC role change; artifact bits do not. Protected environments separate development authority from production deployment.
8. **Deployment:** Rolling, canary or blue/green controls blast radius. Feature flags separate code deployment from feature exposure but need ownership, expiry and safe defaults.
9. **Validation:** Smoke tests prove critical paths; integration/synthetic tests and RED/SLO metrics prove user behavior. Deployment metadata links commit, digest, config and change window.
10. **Recovery:** Stop/pause automatically on predefined evidence. Roll back immutable compute only if schema/config/data remain compatible; otherwise roll forward, restore or reconcile with an incident plan.

## Controls by stage

| Stage | Evidence/control |
|---|---|
| Source | Protected branches, review, signed/traceable changes |
| Build | Pinned dependencies, isolated runner, reproducible artifact |
| Test | Unit, contract, integration, performance/security proportional to risk |
| Artifact | Immutable digest, provenance, SBOM, scan/signature |
| Deploy | Short-lived environment role, change diff, concurrency control |
| Verify | Smoke tests, metrics/logs, automated rollback threshold |

## Testing and security vocabulary

| Control | Detects | Limitation |
|---|---|---|
| Unit test | Logic regression in isolation | Cannot prove integrations |
| Integration/contract | Dependency/interface behavior | Slower; test environment fidelity matters |
| SAST | Source-code weakness patterns | False positives/context gaps |
| DAST | Runtime externally visible weakness | Only exercised paths/configuration |
| SCA | Known vulnerable dependency/license | Database freshness and reachability matter |
| Container scan | Known packages/config in image | Does not prove runtime exploitability |
| Secret scan | Credential-like values | Rotate/revoke real exposure; deletion from latest commit is insufficient |
| Quality gate | Agreed measurable threshold | Bad thresholds encourage gaming |

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

Artifacts and caches are different: an artifact is an immutable trusted output promoted to production; a cache is an optimization and may be rebuilt or poisoned. Registries such as ECR need immutable tags/digests, lifecycle, scan/signature policy, cross-account pull controls and retention that preserves rollback artifacts.

## Terraform delivery lane

```text
PR: fmt -> validate -> module tests -> security/policy -> speculative plan
merge: refresh -> saved plan -> approval -> apply exact plan -> drift/smoke checks
```

Separate plan and apply roles where practical, protect state/plan artifacts, serialize applies, and require a new approval if the reviewed plan is stale or regenerated. Never auto-apply an unexpected destroy to “fix drift.”

## Database migration lane

Use expand/contract:

1. Add backward-compatible schema/index with lock/runtime assessment.
2. Deploy code that tolerates old and new shapes.
3. Backfill in throttled, restartable batches and validate.
4. Switch reads/writes through configuration/feature flag.
5. Remove obsolete code/schema in a later release after rollback window.

Snapshot-before-change is useful but not a substitute for a practical rollback because restoring an entire database can exceed RTO and lose newer writes.

## Pipeline security

Use OIDC federation, least-privilege roles per environment, protected environments/approvals, masked secrets, trusted actions/plugins pinned to immutable versions, isolated ephemeral runners where possible, dependency/provenance controls, and restricted fork behavior. Treat pull requests as untrusted input.

Secrets belong in a managed store and are fetched by the runtime/deployment identity. Redaction is defense in depth; once a secret reaches output, cache or artifact, revoke/rotate and investigate. Self-hosted runners need strong trust segmentation, patching, ephemeral cleanup and network egress controls because build code executes with runner access.

## Post-deployment validation and rollback decision

Validate critical transaction success, error rate, p95/p99 latency, saturation, target/pod readiness, dependency errors and business signals against a baseline. Use a bake window proportional to delayed failure modes. Roll back when the previous version is known safe and data/config are compatible; roll forward when rollback is unsafe or a narrow fix is faster. Always preserve evidence and prevent concurrent releases.

## Interview follow-ups

- **Why not rebuild per environment?** It breaks evidence that the tested bytes equal production bytes.
- **What if a scan finds a critical CVE?** Assess reachability/exposure and policy; block or use approved expiring exception with mitigation, owner and rebuild deadline.
- **How do you deploy without downtime?** Redundant ready capacity, graceful drain, backward-compatible dependencies, incremental traffic and SLO gates.
- **How do you speed up safely?** Parallelize independent tests, cache dependencies safely, use small changes and shift controls earlier; never remove the evidence required for risk.
- **What happens when validation fails?** Stop promotion, protect users, rollback/roll forward according to compatibility, communicate, then root-cause and improve the gate.

## Senior answer

“I separate artifact creation from promotion. A reviewed commit produces one immutable, scanned, traceable artifact. Environment-specific deployment assumes short-lived least-privilege roles, applies policy and approval gates, releases through canary or blue/green, and evaluates service-level signals. Database changes remain backward-compatible. The system blocks concurrent releases and automatically stops or rolls back on predefined evidence.”
