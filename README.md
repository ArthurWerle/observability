# observability

Single-pane observability for the homelab: **logs + metrics + alerts** for every
container on the host, on the LAN only.

| Layer | Tool | What it does |
|-------|------|--------------|
| Dashboards & alerts | **Grafana** | One UI for logs + metrics; e-mails critical failures |
| Logs | **Loki** + **Grafana Alloy** | Alloy auto-discovers every container via the Docker socket and ships logs to Loki (30-day retention) |
| Metrics | **Prometheus** + **cAdvisor** + **node-exporter** | Per-container CPU/mem/net/restarts, host CPU/RAM/disk, plus each app's `/metrics` |

Nothing is exposed outside the LAN.

## Deploy

1. `cp .env.example .env` and fill it in — Grafana admin password and the Gmail
   SMTP **App Password** (see below).
2. Deploy the stack:
   - **Portainer**: Stacks → Add stack → upload this repo (or paste
     `docker-compose.yml`) → add the `.env` variables → Deploy.
   - **CLI**: `docker compose up -d`
3. Open Grafana at `http://<mini-pc-ip>:3000` and log in. The **Homelab Overview**
   dashboard, the Loki/Prometheus datasources, and the e-mail alerts are already
   provisioned.

### Gmail App Password (for alerts)

Gmail with 2-Step Verification does **not** accept your normal password over SMTP.
Create an App Password: Google Account → Security → 2-Step Verification → App
passwords → generate one for "Mail". Put the 16-character value in `SMTP_PASSWORD`.

Test it in Grafana: Alerting → Contact points → `email-arthur` → **Test**.

## How to add a new service

**Logs: nothing to do.** Alloy picks up any new container automatically.

**Metrics (opt-in):** expose a Prometheus `/metrics` endpoint in the service
(reuse the pattern already added to the service in the same language), then add
these labels to its `docker-compose.yml` service, using the **published host
port**:

```yaml
    labels:
      prometheus.io/scrape: "true"
      prometheus.io/hostport: "3005"   # the host-side published port
      prometheus.io/path: "/metrics"   # optional, defaults to /metrics
      environment: "production"         # or "staging"
```

Redeploy the stack in Portainer. Prometheus discovers it from the Docker socket
and scrapes it through the host's published port (no shared network needed), and
it shows up in Grafana within ~30s.

## Alerts included

Provisioned in `grafana/provisioning/alerting/` and delivered by e-mail:

- **Service target DOWN** — a service's `/metrics` stopped responding or its
  container disappeared.
- **Host disk almost full** (> 85% on `/`).
- **Host memory high** (> 90%).
- **Error log spike** — a container logged > 20 error lines in 5 minutes
  (matched from Loki).

Tune thresholds in `grafana/provisioning/alerting/alert-rules.yml`.

## More dashboards

The bundled **Homelab Overview** covers the essentials. For deeper drill-downs,
import these community dashboards (Grafana → Dashboards → New → Import → by ID):

- **1860** — Node Exporter Full (host)
- **14282** — cAdvisor / Docker containers
- Loki logs: explore ad-hoc under Grafana → Explore → Loki with e.g.
  `{compose_project="financer-transactions"} | level="error"`.

## Layout

```
docker-compose.yml            grafana, loki, alloy, prometheus, cadvisor, node-exporter
.env.example                  admin login, Gmail SMTP, retention
prometheus/prometheus.yml     scrape config (cadvisor, node-exporter, docker SD)
loki/loki-config.yml          single-binary Loki, filesystem, 30d retention
alloy/config.alloy            Docker log discovery -> Loki
grafana/
  provisioning/datasources    Prometheus + Loki
  provisioning/dashboards     dashboard provider
  provisioning/alerting       contact point (e-mail), policy, alert rules
  dashboards                  Homelab Overview
```
