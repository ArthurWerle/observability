# Bakes loki-config.yml into the image (see README for why bind-mounts don't
# work under Portainer).
FROM grafana/loki:3.3.2
COPY loki-config.yml /etc/loki/loki-config.yml
