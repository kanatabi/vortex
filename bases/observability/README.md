# observability — reading the estate per environment

Alloy (worker DaemonSet) + Loki + VictoriaMetrics + Grafana (`anor.gofar.id`)
— one shared instance for the WHOLE estate: it lives in the foundation plane
(`environments/foundation/observability`), the Grafana door is
production-owned DNS (unchanged host), and there are no staging/production
twins of any of it.

**Environment attribution.** The estate does NOT run per-environment Alloy
external_labels — one fleet cannot stamp which environment it is watching.
Instead the pod-metrics relabel pipeline copies each pod's `omicron/env`
label into a normalized `env` series label (see `alloy/config.alloy`). The
label is stamped onto every pod template by the env trees' labels
transformers:

| tree                              | omicron/env  |
| --------------------------------- | ------------ |
| environments/foundation/*         | `shared`     |
| environments/staging/*            | `staging`    |
| environments/production/*         | `production` |

So in Grafana (Explore / dashboards / alert rules) prefix or filter any
series with `env="production"` (or `staging`, `shared`) — e.g.
`sum(rate(lumen_http_requests_total{env="staging"}[5m]))`. Pod logs via
the kubelet file tailer keep only `namespace`/`pod` (path-derived, no k8s
labels); co-tenant pods stay distinguishable by the suffixed pod names
(`*-staging` / `-production`).
