# SRE Contract — How to onboard a service onto the SideKick SLO platform

> Owners: Platform SRE. Consumers: every product team. Status: living document.

The observability platform turns standard signals into SLOs, MWMBR burn-rate
alerts, dashboards and routed pages **without per-service code changes**. To
get all of that, a service must satisfy the contract below.

---

## 1. Metric naming (required)

Emit Prometheus metrics on `/metrics` (or any path declared in your
`ServiceMonitor`). Use the OpenMetrics / Prometheus client conventions:

| Signal       | Metric                                  | Type      | Required labels                          |
|--------------|-----------------------------------------|-----------|------------------------------------------|
| Traffic      | `http_requests_total`                   | counter   | `code` (HTTP status), `method`           |
| Latency      | `http_request_duration_seconds_bucket`  | histogram | `code`, `method`, `le`                   |
| Errors       | derived from `http_requests_total{code=~"5.."}` | —   | —                                        |
| Saturation   | `process_cpu_seconds_total`, `process_resident_memory_bytes` | counter / gauge | — |

The bundled `prometheus-example-app` already exports these. If you use a
different language/runtime, pick the idiomatic client library that publishes
the same metric names.

## 2. Required labels (cluster-wide)

Every workload Pod, Service, ServiceMonitor and PrometheusServiceLevel **MUST**
carry:

```yaml
metadata:
  labels:
    app.kubernetes.io/name: <service-name>      # e.g. service-a
    app.kubernetes.io/part-of: <product>        # e.g. sidekick
    team: <team>                                # e.g. team-a — drives paging
    tier: <frontend|backend|data|infra>         # used for SLO grouping
```

`team` is the **routing key** Alertmanager uses to deliver pages. A missing
`team` label = a page that goes to the platform null receiver = nobody
gets woken up. Don't be that service.

## 3. ServiceMonitor

Drop a `ServiceMonitor` next to your `Service`. Two non-negotiable rules:

1. Include `release: kube-prometheus-stack` so the operator picks it up.
2. Relabel the `job` label to the **stable service name** — SLO queries pin to
   `job=<service-name>`, so renaming jobs breaks alerts.

See `applications/service-a/base/servicemonitor.yaml` for the reference.

## 4. PrometheusServiceLevel (the SLO)

Add a `PrometheusServiceLevel` (Sloth CRD) to your overlay. Minimum shape:

```yaml
apiVersion: sloth.slok.dev/v1
kind: PrometheusServiceLevel
metadata:
  name: <service-name>
  labels:
    team: <team>
    tier: <tier>
spec:
  service: <service-name>          # becomes sloth_service label
  labels:
    team: <team>
    tier: <tier>
  slos:
    - name: availability
      objective: 99.5              # %
      description: HTTP 5xx ratio
      sli:
        events:
          error_query: sum(rate(http_requests_total{job="<service-name>",code=~"5.."}[{{.window}}]))
          total_query: sum(rate(http_requests_total{job="<service-name>"}[{{.window}}]))
      alerting:
        name: <ServiceName>Availability
        page_alert:   {labels: {severity: page,   team: <team>}}
        ticket_alert: {labels: {severity: ticket, team: <team>}}
```

Sloth will auto-generate:
- recording rules `slo:sli_error:ratio_rate{5m,30m,1h,6h,1d,3d,30d}`
- the MWMBR burn-rate alerts (fast + slow window, page + ticket)

## 5. Alert routing

Alertmanager routes by the `team` label (see `observability/alertmanager/platform-routing.yaml`).
If your team doesn't have a receiver yet, open a PR adding one to that file —
do **not** invent new label keys.

Annotations the platform expects on every alert:

| Annotation     | Purpose                                                |
|----------------|--------------------------------------------------------|
| `summary`      | one-line page text                                     |
| `description`  | what happened, what threshold, what window             |
| `runbook_url`  | link to your runbook — pages without a runbook are bugs |

## 6. What you get for free

- `SLO — Service Overview` Grafana dashboard, parameterized by `$service`
- `SLO — Registry` dashboard listing every SLO in the cluster
- Multi-window multi-burn-rate paging (5m/1h fast, 30m/6h slow)
- Alertmanager routing to your team receiver
- 30-day error budget tracking

## 7. Checklist before merging

- [ ] Metrics on `/metrics` with required names
- [ ] Labels `team`, `tier`, `app.kubernetes.io/name`, `app.kubernetes.io/part-of`
- [ ] `ServiceMonitor` with `release: kube-prometheus-stack` and `job` relabel
- [ ] `PrometheusServiceLevel` with at least one SLO
- [ ] Receiver exists in `alertmanager/platform-routing.yaml` for your `team`
- [ ] Runbook URL set in alert annotations
