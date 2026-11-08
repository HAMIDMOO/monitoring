# Monitoring Stack Configuration Guide

A step-by-step deployment guide for Node Exporter, NVIDIA DCGM Exporter, Prometheus, and Grafana.

## Prerequisites and Environment Setup

### 1. Install Docker (Ubuntu)

On a new Ubuntu host, open a terminal and run:

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF_DOCKER
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF_DOCKER

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

### 2. Verify Docker

Run:

```bash
sudo docker --version
sudo docker compose version
sudo docker run --rm hello-world
```

### 3. Check NVIDIA GPU (GPU Hosts Only)

Run:

```bash
nvidia-smi
```

If the driver is not installed, install a compatible NVIDIA driver before proceeding.

### 4. Install NVIDIA Container Toolkit (GPU Hosts Only)

Run:

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg2

curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

> Run the Docker restart during a maintenance window if workloads are already running.

### 5. Verify GPU Access in Docker (GPU Hosts Only)

Run:

```bash
sudo docker run --rm --gpus all ubuntu:24.04 nvidia-smi
```

## Node Exporter and NVIDIA DCGM Exporter

### 6. Create the Exporters Directory

Run on each Linux host:

```bash
mkdir -p ~/monitoring/exporters
cd ~/monitoring/exporters
```

### 7. Configure Exporter Listening Address

Create `.env`:

```bash
cat > .env <<'EOF_ENV'
EXPORTER_BIND_IP=127.0.0.1
EOF_ENV
```

### 8. Create Docker Compose Configuration

Open the file:

```bash
nano docker-compose.yaml
```

Add:

```yaml
services:
  node-exporter:
    image: quay.io/prometheus/node-exporter:v1.12.1
    container_name: monitoring-node-exporter
    restart: unless-stopped
    network_mode: host
    pid: host
    volumes:
      - /:/host:ro,rslave
    command:
      - '--path.rootfs=/host'
      - '--web.listen-address=${EXPORTER_BIND_IP:-127.0.0.1}:9100'

  gpu-exporter:
    profiles: ["gpu"]
    image: nvcr.io/nvidia/k8s/dcgm-exporter:4.6.1-4.8.4
    container_name: monitoring-gpu-exporter
    restart: unless-stopped
    network_mode: host
    gpus: all
    command:
      - '--address'
      - '${EXPORTER_BIND_IP:-127.0.0.1}:9400'
```

Save the file.

### 9. Start Node Exporter (All Linux Hosts)

```bash
sudo docker compose config -q
sudo docker compose up -d node-exporter
sudo docker compose ps
```

### 10. Verify Node Exporter

```bash
curl -fsS http://127.0.0.1:9100/metrics -o /tmp/node-exporter.prom

grep -m 3 '^node_cpu_seconds_total' /tmp/node-exporter.prom

grep -m 1 '^node_memory_MemTotal_bytes' /tmp/node-exporter.prom
```

### 11. Start DCGM Exporter (NVIDIA GPU Hosts Only)

Verify GPU access:

```bash
nvidia-smi
sudo docker run --rm --gpus all ubuntu:24.04 nvidia-smi
```

Start both exporters:

```bash
sudo docker compose --profile gpu up -d
sudo docker compose ps
```

### 12. Verify DCGM Exporter

```bash
curl -fsS http://127.0.0.1:9400/metrics -o /tmp/dcgm-exporter.prom

grep -m 5 '^DCGM_FI_DEV_' /tmp/dcgm-exporter.prom
```

### 13. Enable Remote Scraping (Remote Hosts Only)

Allow incoming traffic **only from the Prometheus server** in the host and upstream firewalls before changing the listener address.

For hosts using an active UFW firewall, replace `PROMETHEUS_SERVER_IP`:

```bash
sudo ufw allow from PROMETHEUS_SERVER_IP to any port 9100 proto tcp
```

On GPU hosts, also run:

```bash
sudo ufw allow from PROMETHEUS_SERVER_IP to any port 9400 proto tcp
```

Set the listener address:

```bash
cd ~/monitoring/exporters
sed -i 's/^EXPORTER_BIND_IP=.*/EXPORTER_BIND_IP=0.0.0.0/' .env
```

Apply on CPU-only hosts:

```bash
sudo docker compose up -d --force-recreate node-exporter
```

Apply on GPU hosts:

```bash
sudo docker compose --profile gpu up -d --force-recreate
```

Verify listening ports:

```bash
sudo ss -lntp | grep -E ':(9100|9400)\b'
```

From the Prometheus host, replace `EXPORTER_HOST_IP`:

```bash
curl -fsS http://EXPORTER_HOST_IP:9100/metrics -o /dev/null
```

On GPU hosts:

```bash
curl -fsS http://EXPORTER_HOST_IP:9400/metrics -o /dev/null
```

### 14. Inspect Exporter Logs

```bash
cd ~/monitoring/exporters
sudo docker compose logs --tail=50 node-exporter
```

On GPU hosts:

```bash
sudo docker compose logs --tail=50 gpu-exporter
```

## Prometheus

### 15. Create Prometheus Directory

Run on the monitoring server:

```bash
mkdir -p ~/monitoring/prometheus/secrets
cd ~/monitoring/prometheus
```

### 16. Create Prometheus Configuration

```bash
nano prometheus.yml
```

Add:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['127.0.0.1:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['127.0.0.1:9100']
        labels:
          server: 'local-host'
```

Save the file.

### 17. Create Docker Compose File

```bash
nano docker-compose.yaml
```

Add:

```yaml
services:
  prometheus:
    image: prom/prometheus:v3.13.5
    container_name: monitoring-prometheus
    restart: unless-stopped
    network_mode: host

    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus

    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.listen-address=127.0.0.1:9090'

volumes:
  prometheus-data:
```

Save the file.

### 18. Validate and Start Prometheus

```bash
sudo docker compose config -q

sudo docker run --rm \
  -v "$PWD/prometheus.yml:/etc/prometheus/prometheus.yml:ro" \
  --entrypoint promtool \
  prom/prometheus:v3.13.5 \
  check config /etc/prometheus/prometheus.yml

sudo docker compose up -d prometheus
sudo docker compose ps
```

### 19. Verify Prometheus

```bash
curl -fsS http://127.0.0.1:9090/-/ready

curl -fsSG http://127.0.0.1:9090/api/v1/query \
  --data-urlencode 'query=up'
```

### 20. Add Remote Linux Servers (Optional)

```bash
nano ~/monitoring/prometheus/prometheus.yml
```

Add another target under the existing `node-exporter` job's `static_configs`:

```yaml
      - targets: ['REMOTE_SERVER_IP:9100']
        labels:
          server: 'remote-host'
```

Replace `REMOTE_SERVER_IP` with the target server address.

### 21. Add NVIDIA GPU Targets (Optional)

```bash
nano ~/monitoring/prometheus/prometheus.yml
```

Add a new entry under `scrape_configs`:

```yaml
  - job_name: 'gpu-exporter'
    static_configs:
      - targets: ['GPU_SERVER_IP:9400']
        labels:
          server: 'gpu-host'
```

Replace `GPU_SERVER_IP` with `127.0.0.1` for a local GPU exporter or the remote GPU host address.

### 22. Add Authenticated HTTPS Metrics (Optional)

Create a local secret file:

```bash
cd ~/monitoring/prometheus
sudo install -d -o root -g 65534 -m 750 secrets
sudo install -o root -g 65534 -m 640 /dev/null secrets/service_token

read -rsp 'Metrics bearer token: ' METRICS_TOKEN
echo
printf '%s' "$METRICS_TOKEN" | sudo tee secrets/service_token >/dev/null
unset METRICS_TOKEN
```

Add to the `prometheus` service under `volumes` in `docker-compose.yaml`:

```yaml
      - ./secrets:/etc/prometheus/secrets:ro
```

Add to `scrape_configs` in `prometheus.yml`:

```yaml
  - job_name: 'litellm'
    scheme: https
    metrics_path: /metrics/
    authorization:
      type: Bearer
      credentials_file: /etc/prometheus/secrets/service_token
    static_configs:
      - targets: ['LITELLM_HOSTNAME']
```

Replace `LITELLM_HOSTNAME` with the DNS hostname (without `https://`).

### 23. Apply Configuration Changes

Validate:

```bash
cd ~/monitoring/prometheus
sudo docker compose config -q

sudo docker run --rm \
  -v "$PWD/prometheus.yml:/etc/prometheus/prometheus.yml:ro" \
  -v "$PWD/secrets:/etc/prometheus/secrets:ro" \
  --entrypoint promtool \
  prom/prometheus:v3.13.5 \
  check config /etc/prometheus/prometheus.yml
```

If only `prometheus.yml` changed:

```bash
sudo docker kill --signal=HUP monitoring-prometheus
```

If `docker-compose.yaml` also changed:

```bash
sudo docker compose up -d --force-recreate prometheus
```

### 24. Verify All Targets

```bash
curl -fsS 'http://127.0.0.1:9090/api/v1/targets?state=active' \
  | python3 -c '
import json,sys
for target in json.load(sys.stdin)["data"]["activeTargets"]:
    print(target["labels"].get("job"),
          target["discoveredLabels"].get("__address__"),
          target["health"], target["lastError"])
'
```

## Grafana

### 25. Create Grafana Directories

Run on the monitoring server:

```bash
mkdir -p ~/monitoring/grafana/{provisioning/datasources,dashboards}
cd ~/monitoring/grafana
```

### 26. Configure Prometheus Data Source

```bash
nano provisioning/datasources/prometheus.yml
```

Add:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    uid: prometheus
    type: prometheus
    access: proxy
    url: http://127.0.0.1:9090
    isDefault: true
    editable: true
```

Save the file.

### 27. Create Docker Compose File

```bash
nano docker-compose.yaml
```

Add:

```yaml
services:
  grafana:
    image: grafana/grafana:13.3.0
    container_name: monitoring-grafana
    restart: unless-stopped
    network_mode: host

    environment:
      GF_SERVER_HTTP_ADDR: "127.0.0.1"
      GF_SERVER_HTTP_PORT: "3000"
      GF_SECURITY_ADMIN_USER: "admin"
      GF_USERS_ALLOW_SIGN_UP: "false"
      GF_AUTH_ANONYMOUS_ENABLED: "false"

    volumes:
      - grafana-data:/var/lib/grafana
      - ./provisioning:/etc/grafana/provisioning:ro

volumes:
  grafana-data:
```

Save the file.

### 28. Start Grafana

```bash
sudo docker compose config -q
sudo docker compose up -d grafana
sudo docker compose ps
```

### 29. Verify Grafana

```bash
curl -fsS http://127.0.0.1:3000/api/health
sudo docker compose logs --tail=50 grafana
```

### 30. Access Grafana UI

For a remote workstation, run on the workstation:

```bash
ssh -N -L 3000:127.0.0.1:3000 USER@MONITORING_SERVER_IP
```

Open in the workstation browser:

```text
http://127.0.0.1:3000
```

For a new Grafana installation, sign in using:

```text
Username: admin
Password: admin
```

Change the password when prompted.

### 31. Verify Prometheus Data Source

In Grafana:

```text
Connections → Data sources → Prometheus → Test
```

Then:

```text
Explore → Prometheus → Code → Run query
```

Query:

```promql
up
```

### 32. Import Dashboards from the UI

Keep the dashboard JSON files in the Git repository under:

```text
grafana/dashboards/
```

In Grafana:

```text
Dashboards → New → Import dashboard → Upload dashboard JSON file
```

Select the local JSON file, choose **Prometheus** if prompted, and click **Import**.
