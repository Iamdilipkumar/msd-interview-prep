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

### Users, groups, roles, and Identity Center

- **IAM user:** account-local identity with password and/or access keys. Avoid for workforce access; exceptions need ownership, rotation and monitoring.
- **IAM group:** collection of IAM users used to attach identity policies. Roles cannot be group members, and groups are not principals in resource policies.
- **IAM role:** identity assumed by a trusted principal or AWS service to receive temporary STS credentials.
- **Instance profile:** EC2 attachment container for an IAM role; applications obtain rotating credentials through the SDK credential chain/IMDS.
- **Service role:** role an AWS service assumes to act for you. A service-linked role is predefined/linked to a service lifecycle.
- **IAM Identity Center:** workforce federation and centrally assigned permission sets across AWS accounts. Permission sets provision account roles; the active role session still participates in normal IAM evaluation.

For humans, prefer an external identity provider/Identity Center, MFA, short sessions and role-based account access. For workloads, use the platform’s temporary identity. Access keys are long-lived bearer credentials: avoid them; if unavoidable, never share, scope narrowly, rotate through overlap, detect use and disable the old key after validation.

## Trust versus permission

- A role **trust policy** answers who may assume it and under what conditions.
- Role **permission policies** answer what the assumed session may do.
- The caller also needs permission to call `sts:AssumeRole` when applicable.
- Cross-account access usually needs agreement on both sides; test the precise principal ARN and conditions.

### STS and AssumeRole flow

```text
authenticated caller
   | sts:AssumeRole + optional tags/session policy/external ID
   v
role trust policy evaluates caller and conditions
   |
   v
STS returns temporary access key + secret + session token
   |
   v
role session calls service under effective permission ceilings
```

An external ID helps a third party avoid a confused-deputy problem; it is not a password. OIDC federation instead validates signed claims from an identity provider. For GitHub Actions, constrain the AWS trust policy by expected audience and subject/repository/branch or protected environment, then give the role only deployment permissions.

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

## Memorable policy evaluation

```text
1 Authenticate: who is the exact session principal?
2 Gather: identity + resource policies
3 Cap: boundary + session policy + SCP/RCP
4 Context: conditions, tags, network, MFA, source ARN, encryption context
5 Decide: any applicable explicit DENY -> DENY
           otherwise sufficient applicable ALLOW -> ALLOW
           otherwise -> implicit DENY
```

“Explicit deny > allow > default deny” is a memory hook, not the entire algorithm. An SCP or boundary cannot grant access; each restricts what another policy might allow. Resource-policy behavior varies by principal type and same-account/cross-account context. AWS Organizations, VPC endpoint policies and KMS key policies may add further gates.

## JSON policy reading exercises

### Exercise 1 — resource mismatch

```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::example-artifacts"
}
```

**Result:** It does not allow object reads because `GetObject` requires an object resource such as `arn:aws:s3:::example-artifacts/*`. Bucket actions and object actions use different ARN shapes.

### Exercise 2 — explicit deny condition

```json
{
  "Effect": "Deny",
  "Action": "s3:*",
  "Resource": [
    "arn:aws:s3:::example-artifacts",
    "arn:aws:s3:::example-artifacts/*"
  ],
  "Condition": {"Bool": {"aws:SecureTransport": "false"}}
}
```

**Read it:** Requests using insecure transport are denied even if another policy allows S3. Secure requests are not granted by this statement; they still need an allow.

### Exercise 3 — constrained GitHub federation

```json
{
  "Effect": "Allow",
  "Principal": {"Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"},
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
      "token.actions.githubusercontent.com:sub": "repo:example-org/example-repo:environment:production"
    }
  }
}
```

**Read it:** Only a token whose audience and subject exactly match can enter. Replace placeholders with controlled values; the role’s permission policy separately determines deployment actions.

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

## Ordered failure playbooks

### AssumeRole failure

1. Run `aws sts get-caller-identity`; capture role ARN, account, session and Region context.
2. Confirm the caller may invoke `sts:AssumeRole` on the target.
3. Inspect target trust principal and conditions: external ID, MFA, organization, tags, source identity.
4. Check boundary, session policy and SCP explicit denies.
5. Use CloudTrail error details; fix only the mismatched principal/action/condition.

### S3 cross-account access

1. Identify caller session, bucket/object owner and exact API/ARN.
2. Check caller identity permission and bucket/access-point policy agreement.
3. Check Object Ownership/legacy ACL edge cases, Block Public Access and endpoint policy.
4. For SSE-KMS, check the destination key policy/grant and caller permission.
5. Reproduce one object operation and verify CloudTrail data event/request ID.

### GitHub Actions cannot authenticate to AWS

1. Verify workflow has `id-token: write` and is running in the expected event/environment.
2. Capture non-sensitive OIDC claims: audience, subject, repository/ref/environment.
3. Compare the claims with the AWS OIDC provider and role trust conditions.
4. Confirm role ARN/account and organization guardrails; inspect CloudTrail STS failure.
5. Narrowly correct trust—never broaden it to every repository/branch.

### EC2 role is not working

1. Confirm an instance profile/role is attached and retrieve caller identity with the SDK/CLI.
2. Check IMDSv2 token usage, hop limit, proxy/no-proxy and host firewall.
3. Inspect credential-provider precedence for stale environment/profile credentials.
4. Evaluate role policies, boundary/SCP, endpoint and target resource/KMS policy.
5. Replace/refresh the application session safely and validate the exact API.

### EKS workload permission failure

1. Identify namespace, service account, pod identity association or IRSA annotation, and actual STS caller.
2. For Pod Identity, check agent/association and supported SDK; for IRSA, check OIDC provider, token audience/subject and role trust.
3. Check role permissions plus target resource/KMS/endpoint policies.
4. Ensure node-role credentials are not masking the intended setup.
5. Restart only when credential refresh semantics require it; verify with the workload’s exact call.

### KMS permission failure

1. Capture caller, operation, key ARN/Region/account and encryption context.
2. Check key enabled/pending-deletion state and whether alias resolved to expected key.
3. Evaluate identity policy, key policy, grants, SCP and endpoint policy.
4. Verify service-specific `ViaService`, source and encryption-context conditions.
5. Apply the narrow grant/policy correction and test decrypt/encrypt through the real service.

## MFA, review, and Access Analyzer

Require phishing-resistant MFA where available for workforce and especially privileged paths. MFA conditions can protect sensitive operations but must align with federated sessions and emergency access. IAM Access Analyzer identifies external/public access and can generate policy suggestions from CloudTrail activity; findings and generated policies require human review and representative observation windows.

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
