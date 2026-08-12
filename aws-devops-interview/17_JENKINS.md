# 17 — Jenkins

## Architecture

```text
SCM webhook -> controller schedules -> ephemeral agent executes
                  | plugins/config       | workspace/credentials scope
                  -> artifact + logs + deployment evidence
```

Keep the controller for orchestration, not builds. Use disposable agents, configuration as code, backed-up controller state, tested restore, and a disciplined plugin lifecycle.

## Pipeline principles

- Store Jenkinsfiles with code; use reviewed shared libraries for stable patterns.
- Keep stages resumable/idempotent and use timeouts, retry only transient operations, and clean workspaces.
- Build once and promote immutable artifacts.
- Use credential bindings narrowly; prevent command echo and unsafe interpolation.
- Separate controllers/credentials or strong authorization boundaries according to trust and blast radius.

A **Jenkinsfile** stores pipeline as code. Declarative Pipeline provides a structured `pipeline { agent ... stages ... post ... }` model; Scripted Pipeline offers Groovy flexibility with more room for ungoverned complexity. Stages communicate purpose; parallel branches shorten independent checks; `post` handles cleanup/evidence but must not hide the primary failure.

```groovy
pipeline {
  agent none
  stages {
    stage('Test') {
      parallel {
        stage('Unit') { agent { label 'linux' }; steps { sh './run-unit-tests' } }
        stage('Scan') { agent { label 'linux' }; steps { sh './run-security-scan' } }
      }
    }
    stage('Deploy') {
      agent { label 'trusted-deploy' }
      steps { input message: 'Review artifact and change evidence' }
    }
  }
  post { always { archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true } }
}
```

This generic skeleton intentionally omits credentials and production commands. In a real pipeline, build one immutable artifact and promote it; do not rebuild during the deploy stage.

## Credentials, artifacts, plugins, and shared libraries

- Credential bindings limit exposure but masking is not perfect. Scope by folder/job, disable shell echo, avoid Groovy interpolation and isolate untrusted jobs.
- Archive immutable artifacts in a proper registry/repository; Jenkins build archives need retention and integrity controls.
- Plugins run inside a high-trust controller ecosystem. Minimize, patch, test upgrades and remove abandoned dependencies.
- Shared libraries centralize reviewed stages and policy. Version them, expose a small contract and avoid giving every repository unrestricted Groovy power.
- Webhooks provide low-latency SCM triggers; validate origin/signature as supported and retain periodic reconciliation if missed events matter.

## Failure diagnosis

| Symptom | Check |
|---|---|
| Queue grows | Agent labels/capacity, executor count, cloud provisioning, blocked locks |
| Agent offline | Network/TLS, Java/runtime compatibility, resource pressure, launcher logs |
| Controller slow | Heap/GC, disk latency/space, queue, plugin threads, excessive build work |
| Pipeline hung | Input/lock, external command, timeout, orphaned process, API dependency |
| Credentials exposed | Console masking limitations, shell tracing, archived workspace, plugin behavior |

## Security

Patch controller and agents, minimize/test plugins, enforce SSO/RBAC, CSRF protections and secure agents, isolate untrusted builds, restrict script approvals, encrypt backups, rotate credentials, and audit administrative changes. Plugin sprawl is supply-chain and reliability risk.

## Jenkins versus GitHub Actions

Jenkins offers deep customization and self-hosted control but creates controller/plugin/backup operations. GitHub Actions integrates tightly with GitHub and supports managed runners, but workflow and third-party-action security still require care. Choose based on governance, workload isolation, network access, portability, and operating capacity—not fashion.

| Decision | Jenkins | GitHub Actions |
|---|---|---|
| Control plane | Customer-operated controller | GitHub-managed workflow service |
| Extension | Plugins/shared libraries | Actions/reusable workflows |
| Execution | Customer agents | Hosted and/or self-hosted runners |
| Main risk | Controller/plugin/credential blast radius | Workflow/event/action/runner supply chain |
| Good fit | Deep legacy/custom/on-prem integration | GitHub-centric standardized automation |
