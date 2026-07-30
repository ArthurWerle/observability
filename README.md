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

Deploy from **Git** (Portainer "Repository"), not by pasting the compose. Grafana,
Loki, Alloy and Prometheus are **built** from small Dockerfiles that `COPY` their
config in — see [Why images are built](#why-images-are-built) — so Portainer needs
the repo, and the config files, present to build.

1. **Portainer**: Stacks → Add stack → **Repository**
   - Repository URL: `https://github.com/ArthurWerle/observability`
   - Reference: `refs/heads/main` · Compose path: `docker-compose.yml`
   - Under **Environment variables** add the keys from `stack.env.example`
     (Grafana admin password + Gmail SMTP **App Password**, see below). Portainer
     writes them to a `stack.env` file that the compose loads.
   - Deploy. First deploy builds the four images (a minute or two).
2. **CLI**: `cp stack.env.example stack.env`, edit it, then
   `docker compose up -d --build`.
3. Open Grafana at `http://<mini-pc-ip>:3000` and log in. The **Homelab Overview**
   dashboard, the Loki/Prometheus datasources, and the e-mail alerts are already
   provisioned.

To change any config later, edit the file (e.g. `prometheus/prometheus.yml`) and
redeploy — Portainer rebuilds the image.

### Why images are built

Portainer keeps a stack's files **inside the Portainer container**
(`/data/compose/…`), but a bind-mount source is resolved by the Docker daemon on
the **host**, which can't see that path. It then creates an empty directory there
and the mount fails with *"mounting a directory onto a file"*. Baking each config
into an image via `COPY` sidesteps host-path resolution (the build context is sent
to the daemon by compose), so it deploys reliably under Portainer while keeping the
config files separate and editable in the repo.

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
stack.env.example             admin login, Gmail SMTP, retention (Portainer env vars)
prometheus/
  Dockerfile                  bakes prometheus.yml into the image
  prometheus.yml              scrape config (cadvisor, node-exporter, docker SD)
loki/
  Dockerfile                  bakes loki-config.yml into the image
  loki-config.yml             single-binary Loki, filesystem, 30d retention
alloy/
  Dockerfile                  bakes config.alloy into the image
  config.alloy                Docker log discovery -> Loki
grafana/
  Dockerfile                  bakes provisioning/ + dashboards/ into the image
  provisioning/datasources    Prometheus + Loki
  provisioning/dashboards     dashboard provider (points at /etc/grafana/dashboards)
  provisioning/alerting       contact point (e-mail), policy, alert rules
  dashboards                  Homelab Overview
```

(cAdvisor and node-exporter aren't built — they only mount real host paths
`/`, `/sys`, `/var/run/docker.sock`, which the daemon resolves fine.)
