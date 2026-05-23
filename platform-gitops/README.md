# platform-gitops

The **gitops repo**. Source of truth for everything running in the cluster:
ArgoCD apps, AppProjects, observability stack, and **runtime values** for
every service.

This repo does **not** contain application source code or Helm chart templates.
Those live in the per-service repos (`praveshsignup-lab/service-{a,b,c}`).

```
platform-gitops/
├── argocd-apps/        # ArgoCD Application / ApplicationSet definitions
├── bootstrap/          # ArgoCD itself + AppProjects (one-time apply)
├── observability/      # Prometheus stack values, Alertmanager routing, Sloth SLOs, Grafana dashboards
├── chaos/              # SRE chaos generators (budget burner, etc.)
└── services/           # ENV-SPECIFIC VALUES OVERRIDES — see below
    ├── service-a/
    │   ├── dev/values.yaml
    │   ├── staging/values.yaml
    │   └── prod/values.yaml
    ├── service-b/...
    └── service-c/...
```

## Two-repo workflow

| Change | Where you PR |
|---|---|
| New endpoint / bugfix / log line | service repo |
| New chart template / new label on Pod | service repo |
| Bump replicas in prod | this repo, `services/<svc>/prod/values.yaml` |
| Tighten SLO from 99% → 99.9% in prod | this repo |
| Roll back image tag | this repo (set `image.tag` in env values) |

ArgoCD's `ApplicationSet` per team consumes BOTH sources via Helm
multi-source: the chart from the service repo, the values file from this repo.

```
service repo                 gitops repo (this)
chart/                       services/service-a/prod/values.yaml
  ├── Chart.yaml             ─┐
  ├── values.yaml (defaults)  │   merged at sync time
  └── templates/              │   by ArgoCD's helm renderer
                              ▼
              ArgoCD Application "service-a-prod"
                              ▼
                       cluster `team-a-prod` namespace
```

## SRE observability contract

See [observability/CONTRACT.md](observability/CONTRACT.md). Any service that
follows it gets:

- Auto SLO / MWMBR burn-rate alerts (Sloth)
- Parameterized Grafana dashboards (`slo-overview`, `slo-registry`)
- Slack routing by `team` label (Alertmanager)
- Runbook URL surfaced in every page

## Local validation

```sh
# Render a service against an env values file (mirrors what ArgoCD does):
helm template demo ../service-a/chart \
  -f ../service-a/chart/values.yaml \
  -f services/service-a/prod/values.yaml | less

# Validate any kustomize tree:
kubectl kustomize observability | head
```
