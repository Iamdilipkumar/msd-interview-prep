# 16 — GitHub Actions

## Model

```text
event -> workflow -> jobs -> steps/actions
                  -> runner
                  -> permissions + environment + artifacts/cache
```

Jobs run independently unless linked with `needs`. Artifacts transfer outputs between jobs; caches accelerate reusable dependencies and must not be treated as trusted deployment artifacts.

- **Workflow:** YAML automation stored under `.github/workflows`.
- **Event:** trigger such as push, pull request, manual dispatch, schedule or reusable call.
- **Job:** isolated unit assigned to a runner; dependencies use `needs`.
- **Step/action:** shell command or reusable action within a job.
- **Runner:** managed or self-hosted machine executing untrusted repository instructions.
- **Matrix:** expands a job across versions/platforms/regions; control failure and concurrency deliberately.
- **Environment:** deployment boundary supporting scoped secrets/variables, reviewers and protection rules.

## Secure AWS federation

Prefer GitHub OIDC to stored AWS keys. The workflow needs `id-token: write`; the AWS role trust policy should constrain repository, branch/tag or environment claims, and audience. The role permissions should be environment-specific and narrow.

```yaml
permissions:
  contents: read
  id-token: write
concurrency:
  group: production
  cancel-in-progress: false
```

Pin third-party actions to immutable commit SHAs for higher assurance. Minimize the default `GITHUB_TOKEN` permissions. Do not expose secrets to untrusted forked code or interpolate attacker-controlled values into shell scripts.

## Reusable workflows versus composite actions

- Reusable workflow: shares whole jobs and governance, can define runners/permissions/secrets contract.
- Composite action: bundles steps inside a job and uses that job’s runner/context.

`workflow_call` defines a reusable workflow’s typed inputs, outputs and secrets contract. Callers should grant only required permissions; permissions cannot be safely assumed to expand downstream. Pin reusable workflow/action versions, review changes and centralize sensitive deployment logic in controlled repositories.

## Safe production flow

Use protected environments with reviewers, narrow environment secrets, a saved Terraform plan or immutable image digest, concurrency control, explicit timeouts, provenance, and post-deploy verification. Be precise about `pull_request` versus `pull_request_target`; the latter runs in privileged base-repository context and is dangerous when checking out untrusted code.

```text
pull_request:
  lint + tests + scans + Terraform plan (unprivileged)
        |
merge to protected branch
        |
build/sign/push immutable ECR digest
        |
deploy test -> smoke/integration
        |
production environment approval
        |
OIDC AssumeRole -> deploy digest -> SLO gate -> promote/rollback
```

Branch protection/rulesets should require reviewed PRs, status checks and controlled bypass. An approval is valuable only when the reviewer can see the commit, artifact, plan, risk and rollback—not a blind button press.

## Debugging

Check trigger filters, workflow syntax, job `if`, permissions, environment approval, runner labels/capacity, OIDC claims/trust, action version, shell exit status, artifacts, concurrency, and API rate limits. Enable debug logging only with secret-exposure controls.

## Common traps

- Cache poisoning from broad/attacker-controlled keys.
- Floating action tags.
- Persisted cloud credentials.
- Production environment secrets available during build/test.
- `continue-on-error` hiding a mandatory control.
- Applying a freshly regenerated plan instead of the reviewed plan.
