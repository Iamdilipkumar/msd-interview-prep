# 16 — GitHub Actions

## Model

```text
event -> workflow -> jobs -> steps/actions
                  -> runner
                  -> permissions + environment + artifacts/cache
```

Jobs run independently unless linked with `needs`. Artifacts transfer outputs between jobs; caches accelerate reusable dependencies and must not be treated as trusted deployment artifacts.

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

## Safe production flow

Use protected environments with reviewers, narrow environment secrets, a saved Terraform plan or immutable image digest, concurrency control, explicit timeouts, provenance, and post-deploy verification. Be precise about `pull_request` versus `pull_request_target`; the latter runs in privileged base-repository context and is dangerous when checking out untrusted code.

## Debugging

Check trigger filters, workflow syntax, job `if`, permissions, environment approval, runner labels/capacity, OIDC claims/trust, action version, shell exit status, artifacts, concurrency, and API rate limits. Enable debug logging only with secret-exposure controls.

## Common traps

- Cache poisoning from broad/attacker-controlled keys.
- Floating action tags.
- Persisted cloud credentials.
- Production environment secrets available during build/test.
- `continue-on-error` hiding a mandatory control.
- Applying a freshly regenerated plan instead of the reviewed plan.
