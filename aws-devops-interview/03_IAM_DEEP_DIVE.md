# 03 — IAM Deep Dive

## Authorization model

```text
principal -> authenticate -> request(action, resource, context)
                           -> evaluate all applicable policies
explicit deny > applicable allow > implicit deny
```

An allow may come from identity- or resource-based policy, but boundaries, session policies, organization SCPs/RCPs, resource policies, and service-specific rules change the effective result. Never debug IAM by inspecting only one policy.

## Identities and policies

| Item | Purpose | Senior practice |
|---|---|---|
| User | Long-lived human/service identity | Prefer federation and temporary credentials |
| Role | Assumable identity with temporary credentials | Separate trust policy from permission policy |
| Identity policy | What principal may do | Scope actions/resources/conditions |
| Resource policy | Who may access resource | Restrict principal, source, organization, encryption |
| Permission boundary | Maximum identity-policy permissions | Delegation guardrail; it grants nothing itself |
| SCP | Organization/account maximum | Guardrail; it grants nothing itself |
| Session policy | Narrows an assumed-role session | Useful for brokered, scoped access |

## Trust versus permission

- A role **trust policy** answers who may assume it and under what conditions.
- Role **permission policies** answer what the assumed session may do.
- The caller also needs permission to call `sts:AssumeRole` when applicable.
- Cross-account access usually needs agreement on both sides; test the precise principal ARN and conditions.

## Least privilege workflow

1. Start with required use cases, not `Action: *`.
2. Scope resources and use conditions such as organization, source VPC endpoint, tags, MFA, region, or source ARN when supported.
3. Use separate runtime, deployment, and break-glass roles.
4. Observe CloudTrail/access data; remove unused access.
5. Continuously validate with Access Analyzer and policy simulation, then test a real request.

## Workload identity

| Runtime | Preferred identity pattern |
|---|---|
| EC2 | Instance profile role; protect IMDS and require IMDSv2 where supported |
| ECS | Task role for application; execution role for agent operations |
| EKS | EKS Pod Identity or IRSA per service account; avoid node-role inheritance |
| CI/CD | OIDC federation into narrowly scoped deploy role; avoid stored access keys |
| Cross-account | AssumeRole with constrained trust; external ID for appropriate third parties |

ECS distinction: the **task role** is used by application code; the **task execution role** lets ECS/Fargate pull images, write logs, and retrieve referenced secrets as configured.

## KMS authorization interaction

An IAM allow may be insufficient when a KMS key policy, grant, encryption context, region, alias/ARN selection, or explicit deny blocks the operation. For envelope encryption, applications usually receive a data key protected by a KMS key; KMS need not encrypt the entire payload directly.

## IAM AccessDenied workflow

```text
1 capture principal ARN and request ID
2 confirm action, resource ARN, region, and account
3 inspect CloudTrail event and error context
4 evaluate identity + resource policies
5 evaluate boundary + session policy + SCP/RCP
6 inspect conditions: tags, VPC endpoint, MFA, SourceArn, encryption context
7 for KMS, evaluate key policy/grants too
8 reproduce with policy simulator where supported; apply narrow correction
```

Frequent causes: assumed the wrong role, stale credentials, missing `iam:PassRole`, incorrect ARN shape, resource policy principal mismatch, SCP deny, KMS key policy, endpoint policy, tag condition, or permissions added after a session was issued.

## `iam:PassRole`

PassRole does not assume a role. It permits a principal to configure an AWS service to use a role. Restrict both the role resources and the service receiving it (for example with `iam:PassedToService`) to prevent privilege escalation.

## Common bad answers

- “AdministratorAccess is acceptable temporarily.” Prefer a scoped diagnostic role and expiry.
- “The policy has Allow, so access should work.” Explicit deny or another policy plane may win.
- “A private endpoint makes it secure.” Network reachability and authorization are different controls.
- “Secrets encrypted with KMS are safe in source control.” Ciphertext and configuration can still expose risk; use a secrets service and controlled identity.

## 30-second answer

“I treat IAM as an evaluated request, not a policy document. I first identify the exact session principal, action, resource, and context. I then check identity and resource policies plus boundaries, session policies, organization guardrails, endpoint policies, and—where relevant—KMS key policy. Explicit deny wins. For design, I prefer federation and short-lived roles, split human/deploy/runtime identities, scope resources and conditions, and use CloudTrail and Access Analyzer to refine least privilege.”

## Official references

- [IAM policy evaluation logic](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html)
- [AWS STS temporary credentials](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp.html)
- [ECS IAM roles](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-iam-role-overview.html)
