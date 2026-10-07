# Observability

kube-prometheus-stack + grafana-operator, modeled on [onedr0p/home-ops](https://github.com/onedr0p/home-ops).

## What runs

| Component | Purpose |
|---|---|
| `kube-prometheus-stack/` | Prometheus, Alertmanager, node-exporter, kube-state-metrics (with Flux `gotk_resource_info` metrics) |
| `grafana-operator/` | grafana-operator + a `Grafana` instance, Prometheus + Alertmanager datasources |
| `../../flux/cluster/provider.yaml` + `alert.yaml` | Flux errors → Alertmanager → Pushover (bridge — Flux has no native Pushover provider) |

Alert chain: Prometheus rules → Alertmanager (`AlertmanagerConfig` CR) → Pushover.
Flux chain: notification-controller → Alertmanager → Pushover.
Note the Flux chain now depends on Alertmanager being up; if Alertmanager is down, Flux
failures still show in `task flux:why`.

Also un-blocked by this: `volsync`'s existing `PrometheusRule` + `GrafanaDashboard` start working
(rule/dashboards selectors are now release-agnostic; the dashboard attaches to the Grafana instance
via its `grafana.internal/instance: grafana` label).

## Setup (one time)

1. Pushover credentials: your **User Key** is on the [pushover.net dashboard](https://pushover.net/);
   create an **application** at <https://pushover.net/apps/build> to get an API token.
   Then fill and encrypt the two secrets:
   - `kubernetes/apps/observability/kube-prometheus-stack/app/alertmanager-secret.sops.yaml`
   - `kubernetes/apps/observability/grafana-operator/instance/grafana-admin-secret.sops.yaml`
   Each file has the exact `sops --encrypt --in-place <file>` command in a comment.
   Do not push while any of them is still plaintext.
2. Push. Flux picks up `kubernetes/apps/observability/` automatically (subdirectories with a
   kustomization.yaml are auto-included by the `cluster-apps` Kustomization) and the new files in
   `kubernetes/flux/cluster/` (path has no kustomization.yaml → Flux auto-includes all YAML files).
3. Grafana access (no HTTPRoute yet, intentionally): `kubectl -n observability port-forward deploy/grafana 3000`
   → http://localhost:3000 (admin + anonymous read enabled).

CRDs (`monitoring.coreos.com` etc.) are owned by `bootstrap/helmfile.d/00-crds.yaml` — the chart here
runs with `crds.enabled: false`, so keep that file's version in sync with `app/ocirepository.yaml`
(your renovate config already bumps both).

## Notes / deliberate choices

- No ingress for Prometheus/Alertmanager/Grafana: cluster only exposes `envoy-external` today;
  add an HTTPRoute against an internal gateway later if you want LAN access without port-forward.
- PVCs have no `storageClassName` → OpenEBS default class.
- Alerts: `FluxResourceNotReady` (gotk metric, 30m) and `OomKilled` are custom; default kps rules are on.
  `Watchdog`/`InfoInhibitor` are dropped in Alertmanager (they'd otherwise spam Pushover).
- Warnings do NOT page Pushover — only `severity: critical` and Flux error events (severity
  `error`). Everything else (warnings, Watchdog, InfoInhibitor) falls to the `blackhole` default
  receiver. Tune in `alertmanagerconfig.yaml`.
- Not installed (later, if wanted): smartctl-exporter (disk health), Loki (logs), VictoriaLogs.
