# phil-o-stat

Thermostat / propane / fireplace history for the house: InfluxDB 2, Home
Assistant (ecobee via HomeKit, SwitchBot), Grafana, and two CronJobs.

**The manifests are not in this public repo.** They live in the private repo
`psalwasser/phil-o-stat-historian` (folder `deploy/pi`), because they contain
household details. This folder only tells Flux where to get them:

| File | What |
|---|---|
| `gitrepository.yaml` | Flux source: the private repo over SSH, read-only deploy key in Secret `flux-system/phil-o-stat-auth` |
| `flux-kustomization.yaml` | Applies `./deploy/pi` from that repo (namespace `phil-o-stat`) |

There's no `namespace.yaml` here on purpose. The private repo creates
namespace `phil-o-stat` (with prune disabled), and defining it in both places
would make two Flux Kustomizations fight over it.

## Secrets (created by hand, never in git)

| Secret | Keys |
|---|---|
| `flux-system/phil-o-stat-auth` | `identity`, `identity.pub`, `known_hosts`. Made by `flux create secret git` (see below). |
| `phil-o-stat/thermostat-secrets` | `influx-username`, `influx-password`, `influx-org`, `influx-bucket`, `influx-token`, `grafana-admin-user`, `grafana-admin-password` |
| `phil-o-stat/propane-github` | `token` (fine-grained, read-only GitHub token for the propane data repo) |

```bash
flux --context home create secret git phil-o-stat-auth -n flux-system \
  --url=ssh://git@github.com/psalwasser/phil-o-stat-historian
# -> add the printed public key as a read-only Deploy key on that repo

kubectl --context home create namespace phil-o-stat
kubectl --context home -n phil-o-stat create secret generic thermostat-secrets \
  --from-literal=influx-username=<admin-user> \
  --from-literal=influx-password=<influx-password> \
  --from-literal=influx-org=<org> \
  --from-literal=influx-bucket=<bucket> \
  --from-literal=influx-token=<influx-admin-token> \
  --from-literal=grafana-admin-user=<grafana-user> \
  --from-literal=grafana-admin-password=<grafana-password>
kubectl --context home -n phil-o-stat create secret generic propane-github \
  --from-file=token=<path-to-github-token-file>
```

When migrating, copy `thermostat-secrets` from the old cluster unchanged, so the
restored InfluxDB tokens still match. The full runbook is in the private
repo's `docs/migrate-to-pi.md`.

## URLs (LAN DNS → any node, e.g. 10.0.0.104)

- http://grafana.home.arpa
- http://ha.home.arpa
- http://influx.home.arpa

## Fit notes

- **Images:** all arm64, public Docker Hub, pinned: `influxdb:2.9.1`,
  `homeassistant/home-assistant:2026.10.4`, `grafana/grafana:13.2.5`,
  `python:3.14.9-alpine3.24`.
- **Home Assistant uses `hostNetwork`** (HomeKit / mDNS on the LAN). It binds
  port 8123 on whichever node holds its volume.
- **Storage:** three local-path PVCs: InfluxDB 10Gi, HA 2Gi, Grafana 2Gi. On one
  node's SD card, not replicated, **not in the etcd snapshots**. Back them up
  with `make backup TARGET=pi` in the private repo (automation is a follow-up).
- **Resources:** about 2 GB RAM in total; requests and limits are set on every
  container.
