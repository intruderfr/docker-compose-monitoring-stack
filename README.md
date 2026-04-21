# docker-compose-monitoring-stack

A production-ready, single-command monitoring stack for a Docker host.

Stands up **Prometheus**, **Grafana**, **Alertmanager**, **Node Exporter**,
**cAdvisor**, and **Blackbox Exporter** — pre-wired with datasource
provisioning, a starter dashboard, and a sensible set of alert rules for
host, container, and probe health.

Designed to be a drop-in observability baseline for a home lab, a single
production host, or the start of a larger federated setup.

## Stack

| Component | Image | Purpose | Default port |
|---|---|---|---|
| Prometheus | `prom/prometheus:v2.54.1` | Metrics TSDB + scraping + alerting | `9090` |
| Alertmanager | `prom/alertmanager:v0.27.0` | Alert routing, grouping, silencing | `9093` |
| Grafana | `grafana/grafana:11.2.0` | Dashboards + datasource | `3000` |
| Node Exporter | `prom/node-exporter:v1.8.2` | Host-level metrics (CPU, mem, disk, net) | `9100` (loopback) |
| cAdvisor | `gcr.io/cadvisor/cadvisor:v0.49.1` | Per-container metrics | `8080` (loopback) |
| Blackbox Exporter | `prom/blackbox-exporter:v0.25.0` | HTTP/ICMP/TCP probes | `9115` (loopback) |

All exporters are bound to `127.0.0.1` so Prometheus can reach them on the
docker network while they remain invisible on the host's public interface.

## Quick start

```bash
git clone https://github.com/intruderfr/docker-compose-monitoring-stack.git
cd docker-compose-monitoring-stack

cp .env.example .env
# edit .env — at minimum, change GRAFANA_ADMIN_PASSWORD

docker compose up -d
docker compose ps
```

Then open:

- Grafana: <http://localhost:3000> (`admin` / value of `GRAFANA_ADMIN_PASSWORD`)
- Prometheus: <http://localhost:9090>
- Alertmanager: <http://localhost:9093>

## What you get out of the box

### Scrape targets
Pre-configured in `prometheus/prometheus.yml`:
- Prometheus self-metrics
- Alertmanager
- Node Exporter (host-level)
- cAdvisor (per-container)
- Grafana
- Blackbox HTTP probes for `google.com`, `github.com`, `duckduckgo.com`
- Blackbox ICMP probes for `1.1.1.1`, `8.8.8.8`

Edit `prometheus/prometheus.yml` and run
`docker compose kill -s SIGHUP prometheus` to hot-reload without restart.

### Alert rules
`prometheus/alerts.yml` ships with four rule groups:

- **host.rules** — host down, high CPU (>85% for 10m), high memory (>90% for
  10m), disk >85% / >95%, clock skew.
- **container.rules** — per-container CPU, memory-limit pressure, restart
  loops.
- **blackbox.rules** — probe failures, slow response (>2s), TLS cert
  expiring within 14 days.
- **prometheus.rules** — scrape target missing, config reload failures.

### Alertmanager
Default receiver is a no-op (alerts are logged but not delivered), so the
stack comes up cleanly with no secrets. To wire up delivery, uncomment the
Slack or email block in `alertmanager/alertmanager.yml`, set the matching
env vars in `.env`, and run:

```bash
docker compose kill -s SIGHUP alertmanager
```

### Grafana
- Prometheus datasource auto-provisioned (`grafana/provisioning/datasources/`).
- Dashboard folder auto-provisioned — drop any `*.json` dashboard into
  `grafana/dashboards/` and Grafana picks it up within 30 seconds.
- A starter **Node Exporter Overview** dashboard is included (CPU gauge,
  memory gauge, root disk gauge, CPU over time, network throughput).
- Import popular community dashboards by ID via
  Grafana → Dashboards → New → Import:
  - `1860` — Node Exporter Full
  - `14282` — cAdvisor Exporter
  - `7587` — Blackbox Exporter (HTTP prober)

## File layout

```
.
├── docker-compose.yml
├── .env.example
├── .gitignore
├── LICENSE
├── README.md
├── prometheus/
│   ├── prometheus.yml      # scrape config + alerting target
│   └── alerts.yml          # host/container/probe alert rules
├── alertmanager/
│   └── alertmanager.yml    # routing, grouping, receivers
├── blackbox/
│   └── blackbox.yml        # probe modules (http_2xx, tcp, icmp, dns)
└── grafana/
    ├── provisioning/
    │   ├── datasources/prometheus.yml
    │   └── dashboards/default.yml
    └── dashboards/
        └── node-exporter-overview.json
```

## Operations cheat sheet

```bash
# Tail logs
docker compose logs -f prometheus
docker compose logs -f alertmanager

# Reload Prometheus config without restart
docker compose kill -s SIGHUP prometheus

# Reload Alertmanager config without restart
docker compose kill -s SIGHUP alertmanager

# Validate alert rules before reloading
docker run --rm -v "$PWD/prometheus:/cfg" prom/prometheus:v2.54.1 \
  promtool check rules /cfg/alerts.yml

# Back up Prometheus TSDB (stop first for a clean snapshot)
docker compose stop prometheus
docker run --rm -v monitoring-stack_prometheus_data:/data -v "$PWD:/backup" \
  alpine tar czf /backup/prom-$(date +%F).tgz -C /data .
docker compose start prometheus
```

## Tuning

- **Retention**: `PROMETHEUS_RETENTION` in `.env` (default `15d`). For
  long-term retention, point Prometheus at a remote-write target like
  Thanos, Mimir, or VictoriaMetrics — this stack is deliberately
  single-node.
- **Scrape interval**: `global.scrape_interval` in
  `prometheus/prometheus.yml` (default `15s`).
- **Disk footprint**: rule of thumb is ~1–2 bytes per sample after
  compression. 15s scrape × 15 day retention × N series × 6 is a rough
  upper bound in bytes.

## Security notes

- Change `GRAFANA_ADMIN_PASSWORD` before exposing Grafana on anything
  beyond localhost.
- Node Exporter, cAdvisor, and Blackbox Exporter are bound to `127.0.0.1`
  in the compose file — do not rebind them to `0.0.0.0` on a public host
  without putting an authenticated reverse proxy in front.
- If you expose Grafana or Prometheus over the internet, put them behind
  a reverse proxy with TLS and SSO.

## Troubleshooting

- **Prometheus target shows `DOWN`**: open <http://localhost:9090/targets>
  and check the error. Most common causes are a typo in the scrape config
  or a service that isn't listening yet.
- **Grafana says "Datasource not found"**: confirm the provisioning file
  was mounted — `docker compose exec grafana ls /etc/grafana/provisioning/datasources`.
- **No container metrics**: cAdvisor needs `/var/lib/docker` and `/sys`
  mounted, which the compose file already does. On SELinux-enforcing
  distros you may need `:z` labels on the bind mounts.
- **ICMP probes failing**: some hosts drop outbound ICMP. Swap the
  `icmp` module for a `tcp_connect` on port 443 to the same host.

## License

MIT. See [LICENSE](./LICENSE).

## Author

**Aslam Ahamed** — Head of IT @ Prestige One Developments, Dubai.
Connect on [LinkedIn](https://www.linkedin.com/in/aslam-ahamed/).
