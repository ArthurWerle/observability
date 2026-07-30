# Bakes the provisioning tree (datasources, dashboards provider, alerting) and
# the dashboard JSON into the image, so Portainer doesn't need to bind-mount
# them from the host (see README).
FROM grafana/grafana:11.4.0
COPY provisioning /etc/grafana/provisioning
COPY dashboards /etc/grafana/dashboards
