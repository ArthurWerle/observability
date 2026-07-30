# Bakes prometheus.yml into the image so Portainer doesn't need to bind-mount it
# from the host (see README: Portainer keeps stack files inside its own
# container, out of the Docker daemon's reach).
FROM prom/prometheus:v3.1.0
COPY prometheus.yml /etc/prometheus/prometheus.yml
