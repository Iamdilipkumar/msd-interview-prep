# 14 — Terraform on AWS

## Execution model

```text
configuration + variables + provider APIs + current state
                  -> refresh/plan -> dependency graph -> apply -> new state
```

Terraform reconciles declared configuration using state and provider APIs. State maps resource addresses to remote objects and can contain sensitive data; treat it as critical infrastructure data.

## Production layout

- Small, cohesive root modules per lifecycle/blast radius—not one state for an entire enterprise.
- Reusable versioned child modules with clear inputs, outputs, constraints, and tests.
- Separate state and credentials by environment/account; do not use CLI workspaces as a security boundary.
- CI assumes short-lived AWS roles, produces a reviewed saved plan, and applies that exact artifact under controls.

## S3 backend

Use encrypted S3, Bucket Versioning, least-privilege access, access logging/audit, and state locking. Current Terraform supports S3 native locking with `use_lockfile = true`; DynamoDB-based locking is deprecated. Confirm capabilities for the Terraform version in use.

```hcl
terraform {
  backend "s3" {
    bucket       = "example-terraform-state"
    key          = "platform/network/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

Do not hardcode credentials or sensitive backend configuration. Bootstrap the backend separately and protect deletion/recovery paths.

## Safe pipeline

```text
fmt -> validate -> lint/security/test -> plan -> human/policy gate
    -> apply saved plan -> smoke test -> record evidence
```

Plan output may contain secrets. Restrict artifact/log access and retention. Never run unreviewed speculative plans with production credentials.

## Drift

Detect with scheduled read-only plans and cloud configuration controls. Decide whether the console change was authorized and should be imported into code, or whether code should restore declared state. Do not automatically overwrite emergency changes without understanding them.

## Partial apply

Terraform records successful operations and returns an error for the failed remainder. Freeze competing applies; preserve logs/state; inspect cloud reality, state, and a fresh plan; fix the root cause; then re-plan and apply. Do not blindly rerun or edit state. Import/move/remove state entries only after backup and proof of ownership.

## State corruption/recovery

1. Stop all writers and preserve local/remote state plus logs.
2. Verify backend/lock, serial/lineage, and S3 object versions.
3. Compare state with real resources using read-only queries and `terraform plan -refresh-only` as appropriate.
4. Restore a known-good version or use precise state/import operations.
5. Generate a normal plan and obtain review before applying.
6. Correct access, locking, versioning, and concurrency controls.

`force-unlock` is appropriate only after proving the lock owner is dead and no apply is running. An incorrect unlock enables concurrent writers.

## Refactoring safely

Use `moved` blocks for address changes, `import` blocks/import for existing resources, and explicit version upgrades. Review destroy/recreate indicators such as “forces replacement.” Use lifecycle controls sparingly; `ignore_changes` can conceal dangerous drift.

## Module design

Prefer opinionated modules for stable organizational patterns. Validate inputs, expose only necessary outputs, pin compatible provider/module versions, document breaking changes, and test examples. Avoid a “mega-module” with dozens of booleans and hidden coupling.

## Secrets

`sensitive = true` redacts selected CLI output; it does not remove the value from state. Prefer references/ARNs and runtime retrieval. Avoid secret values in variables, plans, state, outputs, and generated user data wherever possible.

## Common bad answers

- “Terraform state is just a cache.” It is the mapping Terraform needs to manage resources.
- “Run apply again until green.” First reconcile partial success and root cause.
- “Use `-target` for normal deployment.” It is an exceptional recovery tool and can create incomplete convergence.
- “Store access keys in CI secrets.” Prefer OIDC federation and short-lived roles.

## Official reference

- [Terraform S3 backend](https://developer.hashicorp.com/terraform/language/backend/s3)
