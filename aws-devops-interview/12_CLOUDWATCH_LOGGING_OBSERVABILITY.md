# 12 — CloudWatch, Logging, and Observability

## Signals and purpose

| Signal | Answers |
|---|---|
| Metrics | Is behavior changing and where is saturation? |
| Logs | What discrete event/error occurred? |
| Traces | Where did request time and failure propagate? |
| Events | What state/configuration/deployment changed? |

Use service-level **RED** (rate, errors, duration) and resource **USE** (utilization, saturation, errors). Dashboards without actionable thresholds and ownership are decoration.

## Alarm design

Alarm on user impact and leading capacity signals. Define evaluation period, missing-data behavior, statistic/percentile, dimensions, actions, runbook, severity, and owner. Composite alarms reduce noise. Test alarms and notification routes.

## Logging pipeline

```text
source -> structured collection -> centralized account/store -> search/detect/archive
         redact + encrypt         retention + access          cost controls
```

Include timestamp, severity, service, environment, version, request/trace ID, and non-sensitive context. Never log secrets/tokens. Avoid unbounded high-cardinality metric dimensions.

## Incident correlation

Begin with impact and change timeline. Compare affected/unaffected slices by AZ, version, endpoint, dependency, or tenant. Move from SLO to service metrics, traces, logs, then infrastructure. Preserve request IDs and time synchronization.

## Cost controls

Set log retention, filter at source carefully, sample high-volume traces, use metric filters selectively, tier/archive when justified, and track ingest/query spend. Do not reduce the telemetry needed to diagnose high-risk systems without an explicit risk decision.

## Common traps

- Averages hide tail latency; inspect p95/p99 and distribution.
- “No alarm” may mean missing data was treated as good.
- CPU below 50% does not prove health.
- Metrics indicate correlation; traces and profiles help locate causality.

## 30-second answer

“I define observability from service objectives. I alarm on rate, errors, duration, and saturation; propagate correlation IDs across structured logs and traces; record deployment/configuration events; and centralize access with retention and redaction. During incidents I start with user impact and timeline, compare affected slices, follow the critical path, and make one evidence-backed mitigation at a time.”
