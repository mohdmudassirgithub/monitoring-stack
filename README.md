# Monitoring, Alerting & Centralized Logging Stack

A Prometheus, Grafana, Loki and Promtail stack for monitoring Linux hosts and
Docker containers, with alerting on resource usage and service availability.

Everything runs through a single `docker compose up`.

## What it does

- Collects host metrics (CPU, memory, disk, filesystem, network) via **Node Exporter**
- Collects per-container metrics via **cAdvisor**
- Scrapes both into **Prometheus** with a 15 day retention window
- Ships system, Nginx and Docker container logs into **Loki** through **Promtail**
- Pre-provisions **Grafana** with Prometheus and Loki as datasources
- Fires alert rules for CPU, memory, disk, host down, container restart loops
  and service availability

## Stack

| Component | Role |
|---|---|
| Prometheus | metrics storage, scraping, alert rule evaluation |
| Grafana | dashboards and log exploration |
| Loki | log aggregation |
| Promtail | log shipping from host and containers |
| Node Exporter | host-level metrics |
| cAdvisor | container-level metrics |

## Running it

```bash
git clone https://github.com/mohdmudassirgithub/monitoring-stack
cd monitoring-stack

cp .env.example .env        # set a Grafana admin password
docker compose up -d
```

| Service | URL |
|---|---|
| Grafana | http://localhost:3000 |
| Prometheus | http://localhost:9090 |
| Loki | http://localhost:3100 |
| cAdvisor | http://localhost:8080 |

Check that every target is up at **Prometheus → Status → Targets**.

## Alert rules

Defined in `prometheus/alert-rules.yml`.

| Alert | Condition | For | Severity |
|---|---|---|---|
| HighCPUUsage | CPU above 85% | 5m | warning |
| HighMemoryUsage | Memory above 90% | 5m | warning |
| DiskSpaceLow | Less than 10% free | 10m | critical |
| HostDown | Node Exporter unreachable | 2m | critical |
| ContainerRestartLoop | More than 3 restarts in 15m | 5m | warning |
| ContainerHighMemory | Container above 90% of its memory limit | 5m | warning |
| ServiceDown | Application target unreachable | 2m | critical |

After editing rules, reload without a restart:

```bash
curl -X POST http://localhost:9090/-/reload
```

## Querying logs

In Grafana, open **Explore** and select the Loki datasource.

```logql
{job="nginx"} |= "500"                  # Nginx 500s
{container="payment-api"} |= "ERROR"     # container errors
{job="varlogs"} |= "Failed password"     # failed SSH logins
```

## Adding a host

Add the host's Node Exporter target to `prometheus/prometheus.yml`:

```yaml
  - job_name: node-exporter
    static_configs:
      - targets: ['node-exporter:9100', '10.0.0.12:9100']
        labels:
          host: app-server-02
```

Then reload Prometheus with the curl command above.

## Notes

Configuration is based on a monitoring setup I built and ran for a fintech
payment platform. Hostnames, targets and credentials here are placeholders —
no company-specific details are included.
