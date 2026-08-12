# 13 — Secrets, KMS, and Security

## Defense layers

```text
organization/account -> identity -> network -> workload -> data -> detection/recovery
```

No single “secure service” replaces layered controls, ownership, monitoring, and response.

## Secrets

Use Secrets Manager for managed secret lifecycle/rotation and Parameter Store for hierarchical configuration and secret use cases matching its capabilities. Applications retrieve at runtime using workload identity. Cache cautiously, rotate with backward-compatible steps, restrict resource policies, and audit access. Never place secrets in Git, images, user data, Terraform values, logs, or CI output.

## KMS mental model

KMS keys protect data keys in envelope encryption. Authorization may involve identity policy, key policy, grants, conditions/encryption context, organization controls, and service integration. KMS keys are regional resources; multi-Region keys have specific replication semantics and are not a universal DR shortcut.

| Choice | Consider |
|---|---|
| AWS-owned key | Lowest customer control/operation |
| AWS-managed key | Service-managed policy/lifecycle |
| Customer-managed key | Policy, rotation, audit, separation, cross-account control and cost |

## Rotation

A safe secret rotation commonly stages a new value, updates the backing system, validates it, marks it current, and later retires the old value. Applications must refresh and tolerate overlap. Rotation success means clients still authenticate, not merely that a Lambda function returned success.

## Security services by question

| Question | Service examples |
|---|---|
| What API changed? | CloudTrail |
| What configuration violates policy? | AWS Config |
| What threats/anomalies are detected? | GuardDuty, Security Hub findings |
| What data is sensitive? | Macie for supported S3 discovery use cases |
| What vulnerabilities are present? | Inspector |

## Compromised credential response

1. Contain: disable/revoke/deny the affected session path without destroying evidence.
2. Identify scope using CloudTrail and service logs; preserve evidence.
3. Rotate dependent credentials and remove persistence/backdoors.
4. Restore trusted configuration/data; validate integrity.
5. Improve federation, session duration, least privilege, detection, and playbooks.

## KMS AccessDenied

Confirm caller session, ciphertext/key region/account, operation, key state, identity policy, key policy/grant, encryption context, VPC endpoint policy, SCP, and service principal conditions. Avoid broadening the key policy as a first move.

## Memory hook

**Secret = value lifecycle. KMS = key authorization. IAM = caller authorization.** All may be involved in one failed request.
