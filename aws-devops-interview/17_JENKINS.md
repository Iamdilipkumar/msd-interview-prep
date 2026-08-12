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
