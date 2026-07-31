# Node Exporter, cAdvisor, and Grafana Dashboards
>Prometheus is running and we can query metrics. But right now it is only monitoring itself. In production, we need to monitor two critical things: the **host machine** (CPU, memory, disk, network) and the **Docker containers** running on it.

Today we add Node Exporter for host metrics, cAdvisor for container metrics, and set up Grafana to visualize everything in dashboards instead of raw PromQL.

## Table of Contents
1. [Node Exporter - Host Metrics](#1-add-node-exporter-for-host-metrics)
2. [cAdvisor - Container Metrics](#2-add-cadvisor-for-container-metrics)
3. [Set Up Grafana](#3-set-up-grafana)
4. [Build Your First Dashboard](#4-build-your-first-dashboard)
5. [Auto-Provision Datasources with YAML](#5-auto-provision-datasources-with-yaml)
6. [Import Community Dashboards](#6-import-a-community-dashboard)

## 1. Add Node Exporter for Host Metrics
Node Exporter exposes Linux system metrics - CPU, memory, disk, filesystem, network - in Prometheus format via a `/metrics` endpoint.

### Add Node Exporter service
Update our `docker-compose.yml` from Day 73:
>Insert under `services:` block
```yaml
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    restart: unless-stopped
```

### Volume Mounts Explained:
| Mount | Source | Purpose |
|---|---|---|
| `/proc` | Kernel & process info | CPU stats, memory info, process list |
| `/sys` | Hardware & driver details | Device stats, cgroup data |
| `/` | Root filesystem | Disk usage across all mount points |
 
All three are mounted read-only (`:ro`) - Node Exporter never modifies system files.

### Update `prometheus.yml`
Prometheus must know to scrape Node Exporter. Add it as a scrape target in `prometheus.yml`:
```yaml
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]
```

### Restart the stack
Bring up all containers with the new service.
```bash
docker compose up -d
```
- Wait for containers to start

### Verify Node Exporter
Check if Node Exporter is exposing metrics.
```bash
curl http://localhost:9100/metrics | head -20
```
- Should see metric lines like `# HELP node_cpu_seconds_total`

### Check Prometheus Targets
Ensure Prometheus is scraping Node Exporter.
- Open Prometheus UI → `http://localhost:9090/targets`
- Node Exporter should show **UP**

### Run sample queries
Test host metrics in Prometheus:
```promql
# CPU: percentage of time spent idle (per core)
node_cpu_seconds_total{mode="idle"}

# Memory: total vs available
node_memory_MemTotal_bytes
node_memory_MemAvailable_bytes

# Memory usage percentage
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Disk: filesystem usage percentage
(1 - node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100

# Network: bytes received per second
rate(node_network_receive_bytes_total[5m])
```

---

## 2. Add cAdvisor for Container Metrics
cAdvisor (Container Advisor) monitors resource usage and performance characteristics of every running Docker container, exposing the data as Prometheus metrics.

### Update docker-compose.yml
Open our `docker-compose.yml` and add this new service block:
```yaml
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    restart: unless-stopped
```

### Volume mounts explained:
| Mount | Purpose |
|---|---|
| `/var/run/docker.sock` | Lets cAdvisor discover and query running containers |
| `/sys` | Kernel-level container stats via cgroups |
| `/var/lib/docker/` | Container filesystem layer information |
 
All mounts are **read-only** - cAdvisor never modifies Docker state.

### Add Prometheus scrape target
Edit our `prometheus.yml` and add cAdvisor as a Prometheus scrape target:
```yaml
  - job_name: "cadvisor"
    static_configs:
      - targets: ["cadvisor:8080"]
```
- This tells Prometheus to scrape metrics from cAdvisor.

### Restart the stack
```bash
docker compose up -d
```

Check containers:
```bash
docker compose ps
```
- We should see `cadvisor` running alongside Prometheus, Node Exporter, Grafana, and notes-app.

### Verify cAdvisor UI
Open our browser → `http://localhost:8080`
- Click **Docker Containers** → we’ll see per-container CPU, memory, filesystem stats.
This is a quick sanity check before Prometheus queries.

### Run Prometheus queries
Go to Prometheus UI (`http://localhost:9090`) and try:
```promql
# CPU usage per container (in seconds)
rate(container_cpu_usage_seconds_total{name!=""}[5m])

# Memory usage per container
container_memory_usage_bytes{name!=""}

# Network received bytes per container
rate(container_network_receive_bytes_total{name!=""}[5m])

# Which container is using the most memory?
topk(3, container_memory_usage_bytes{name!=""})
```

> Note: The `{name!=""}` filter removes aggregated/system-level entries and shows only named containers.

### What is the difference between Node Exporter and cAdvisor? When would you use each?
| Tool	| Scope	Use | Case |
| --- | --- | --- |
| Node Exporter |	Host-level metrics (CPU, memory, disk, network of the machine)	| Monitor server health |
| cAdvisor	| Container-level metrics (CPU, memory, network per container)	| Monitor workloads running inside Docker |

We need both: a healthy host can still have a runaway container, and a healthy container profile can still hide a failing disk.

## 3. Set Up Grafana
Grafana is the visualisation layer. It connects to Prometheus (and later Loki) as a datasource and provides dashboards, alerting, and an exploration UI - no PromQL knowledge required for day-to-day use.

### Add Grafana service
Open our `docker-compose.yml` and add this block:
```yaml
  grafana:
    image: grafana/grafana-enterprise:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    restart: unless-stopped
```
- This defines Grafana as a container, exposes it on port **3000**, and sets up a persistent volume for dashboards/config.

### Add Grafana volume
At the bottom of our `docker-compose.yml`, ensure both volumes are declared:
```yaml
volumes:
  prometheus_data:
  grafana_data:
```
- Without the named volume, dashboard configurations and datasource settings are lost on every container restart.

### Restart the stack
```bash
docker compose up -d
```

Check containers:
```bash
docker compose ps
```
- We should see **grafana** listed alongside Prometheus, Node Exporter, cAdvisor, and notes-app.

### Access Grafana and Connect Prometheus
1. Open `http://localhost:3000`
2. Log in with `admin` / `admin123`
3. Go to **Connections → Data Sources → Add data source**
4. Select **Prometheus**
5. Set the URL to `http://prometheus:9090`

> Use the container name `prometheus`, not `localhost`. Containers communicate over Docker's internal network - `localhost` inside the Grafana container refers to Grafana itself, not Prometheus.
 
6. Click **Save & Test** - we should see: *"Successfully queried the Prometheus API"*

## 4. Build Your First Dashboard
Create a dashboard that gives an at-a-glance view of host and container health.

### Create a new dashboard
- In Grafana UI (`http://localhost:3000`), log in as `admin/admin123`.
- Go to **Dashboards > New Dashboard > Add Visualization**.
- Select **Prometheus** as the datasource.

### Panel 1: CPU Usage %
- **Query**:
  ```promql
  100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
  ```
- **Visualization**: Gauge
- **Title**: `CPU Usage %`
- **Thresholds**:
  - Green < 60
  - Yellow < 80
  - Red ≥ 80

This shows overall CPU utilization across cores.

### Panel 2: Memory Usage %
- **Query**:
  ```promql
  (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
  ```
- **Visualization**: Gauge
- **Title**: `Memory Usage %`

This shows how much memory is consumed vs available.

### Panel 3: Container CPU Usage
- **Query**:
  ```promql
  rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100
  ```
- **Visualization**: Time series
- **Title**: `Container CPU Usage`
- **Legend**: `{{name}}`

Each container’s CPU usage plotted over time.

### Panel 4: Container Memory Usage
- **Query**:
  ```promql
  container_memory_usage_bytes{name!=""} / 1024 / 1024
  ```
- **Visualization**: Bar chart
- **Title**: `Container Memory (MB)`
- **Legend**: `{{name}}`

Shows per-container memory usage in MB.

### Panel 5: Disk Usage %
- **Query**:
  ```promql
  (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100
  ```
- **Visualization**: Stat
- **Title**: `Disk Usage %`

Displays root filesystem usage percentage.

### Save the dashboard
- Click **Save Dashboard**.
- Name it: `DevOps Observability Overview`.

## 5. Auto-Provision Datasources with YAML
In production, we never configure datasources by clicking through the UI. Provisioning via YAML files makes the setup repeatable, version-controllable, and CI/CD-friendly.

### Create the Provisioning Directory Structure
In our project root:
```bash
mkdir -p grafana/provisioning/datasources
mkdir -p grafana/provisioning/dashboards
```
This sets up the directory structure Grafana expects for provisioning.

### Create `datasources.yml`
`grafana/provisioning/datasources/datasources.yml`:
```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
```
This defines Prometheus as the default datasource.
- `access: proxy` — Grafana queries Prometheus through its own backend, not the browser
- `isDefault: true` — this datasource is pre-selected when creating new panels
- `editable: false` — prevents accidental modification via the UI

### Update Grafana service in `docker-compose.yml`
Modify our Grafana service block to mount the provisioning directory:
```yaml
  grafana:
    image: grafana/grafana-enterprise:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    restart: unless-stopped
```
This ensures Grafana reads our YAML provisioning files at startup.

### Restart Grafana
```bash
docker compose up -d grafana
```
Check logs:
```bash
docker logs grafana | grep "Datasource"
```
We should see Grafana loading the Prometheus datasource automatically.

### Verify in Grafana UI
- Open `http://localhost:3000`.
- Go to **Connections > Data Sources**.
- Prometheus should already be listed - no manual setup required.

### Why is provisioning datasources via YAML better than configuring them manually through the UI?
- **Repeatable**: Same config works across environments (dev, staging, prod).
- **Version-controlled**: Stored in Git, changes are tracked.
- **Automated**: No manual clicks, ideal for CI/CD pipelines.
- **Consistent**: Prevents human error and ensures every environment has identical setup.

## 6. Import a Community Dashboard
The Grafana community maintains thousands of pre-built dashboards. Rather than building panels from scratch, import the two most widely used dashboards for this stack.

### Open Grafana Dashboards
- Log in to Grafana (`http://localhost:3000`).
- Go to **Dashboards > New > Import**.

### Import Node Exporter Full Dashboard
- Enter **Dashboard ID: 1860**.
- Select your **Prometheus datasource**.
- Click **Import**.

This is the community standard for host metrics. It includes panels for CPU utilisation, memory pressure, disk I/O, network throughput, and more - all built on the `node_` metrics exposed by Node Exporter.

### Import Docker Monitoring via cAdvisor
- Again, go to **Dashboards > New > Import**.
- Enter **Dashboard ID: 193**.
- Select **Prometheus** as the datasource.
- Click **Import**.

This dashboard focuses on container-level stats using `container_` metrics from cAdvisor. We’ll see per-container CPU, memory, and network usage.

### Explore the dashboards
- Navigate through both dashboards.
- Compare **Node Exporter Full (1860)** for host health vs **Docker Monitoring (193)** for container workloads.
- These dashboards give you immediate visibility without building panels from scratch.

### Verify all services running
```bash
docker compose ps
```
All five services should show **Up**:
| Container | Port | Purpose |
|---|---|---|
| `prometheus` | 9090 | Metrics storage and PromQL |
| `node-exporter` | 9100 | Host-level metrics |
| `cadvisor` | 8080 | Container-level metrics |
| `grafana` | 3000 | Dashboards and visualisation |
| `notes-app` | 8000 | Sample application (from Day 73) |