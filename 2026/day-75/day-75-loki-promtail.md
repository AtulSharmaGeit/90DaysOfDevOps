# Log Management with Loki and Promtail
>Metrics tell us **_what_** is broken. Logs tell us **_why_**. Yesterday we built the metrics pipeline with Prometheus, Node Exporter, cAdvisor, and Grafana. This guide adds the second pillar of observability -- logs.

We will set up Grafana Loki (a log aggregation system built by the Grafana team) and Promtail (the agent that ships logs to Loki). By the end, Grafana will display metrics and logs side by side.

## Table of Contents 
1. [Understanding the Logging Pipeline](#1-understand-the-logging-pipeline)
2. [Add Loki to the Stack](#2-add-loki-to-the-stack)
3. [Add Promtail to Collect Container Logs](#3-add-promtail-to-collect-container-logs)
4. [Add Loki as a Grafana Datasource](#4-add-loki-as-a-grafana-datasource)
5. [Query Logs with LogQL](#5-query-logs-with-logql)
6. [Correlate Metrics and Logs in 
Grafana](#6-correlate-metrics-and-logs-in-grafana)

## 1. Understand the Logging Pipeline
Before writing any configuration, understand how the pieces connect:
```
[Docker Containers]
        │
        │  write JSON logs to /var/lib/docker/containers/
        ▼
   [Promtail]
        │
        │  reads log files · adds labels · pushes to Loki
        ▼
     [Loki]
        │
        │  stores logs · indexes by labels only
        ▼
   [Grafana]
        │
        │  queries Loki with LogQL · displays log streams
        ▼
     [You]
```
 
### Why Loki Doesn't Index Full Text ?
**Ans.** Most log systems (Elasticsearch, Splunk) index every word in every log line, enabling fast full-text search at the cost of significant storage and CPU overhead.
 
Loki takes a different approach — it **only indexes labels** (container name, job, filename). The log content itself is compressed and stored as-is. This makes Loki dramatically cheaper to operate and simpler to scale, at the trade-off of slower ad-hoc text searches on very large datasets.
 
Think of it as "Prometheus for logs" — the same label-based data model, the same philosophy of keeping the storage layer lean.
 
| | Elasticsearch / Splunk | Loki |
|---|---|---|
| **Indexes** | Full log content | Labels only |
| **Storage cost** | High | Low |
| **Query speed** | Fast for any text | Fast for label queries, slower for content scans |
| **Operational complexity** | High | Low |
| **Best for** | Large-scale text search | Label-filtered log streams |

## 2. Add Loki to the Stack
### Create a Loki Config Directory
We’ll keep Loki’s configuration clean and separate.
```bash
mkdir -p loki
```

### Write the Loki Config File
Create `loki/loki-config.yml`:
```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory
  replication_factor: 1
  path_prefix: /loki

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

storage_config:
  filesystem:
    directory: /loki/chunks
```
**Key settings:**
| Field | Value | Purpose |
|---|---|---|
| `auth_enabled` | `false` | Single-tenant mode - no authentication required |
| `http_listen_port` | `3100` | Port Loki listens on |
| `store` | `tsdb` | Time-series database indexing for log metadata |
| `object_store` | `filesystem` | Log chunks stored locally on disk |
| `replication_factor` | `1` | Single instance - appropriate for development |
| `storage_config.directory` | `/loki/chunks` | Where log chunks are written inside the container |

### Add Loki Service to Docker Compose
Open our `docker-compose.yml` and add this service:
```yaml
  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yml:/etc/loki/loki-config.yml
      - loki_data:/loki
    command: -config.file=/etc/loki/loki-config.yml
    restart: unless-stopped
```

### Add Loki Volume
At the bottom of our `docker-compose.yml`, add `loki_data` alongside the existing volumes:
```yaml
volumes:
  prometheus_data:
  grafana_data:
  loki_data:
```
- This ensures Loki has persistent storage.

### Start Loki:
```bash
docker compose up -d loki
```
- This starts Loki in detached mode.

### Verify Loki is running:
```bash
curl http://localhost:3100/ready
```
Expected output:
```Code
ready
```
- That means Loki is healthy and ready to accept logs.

## 3. Add Promtail to Collect Container Logs
Promtail is the log collection agent. It tails Docker container log files on the host, enriches them with labels, and pushes them to Loki.

### Create a Promtail Config Directory
Keep configs organized:
```bash
mkdir -p promtail
```

### Write the Promtail Config File
Create `promtail/promtail-config.yml`:
```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    static_configs:
      - targets:
          - localhost
        labels:
          job: docker
          __path__: /var/lib/docker/containers/*/*-json.log
    pipeline_stages:
      - docker: {}
```
**Key settings:**
| Field | Purpose |
|---|---|
| `http_listen_port: 9080` | Promtail's own status and metrics endpoint |
| `positions.filename` | Tracks which log lines have already been shipped — acts as a bookmark to survive restarts |
| `clients.url` | Loki's push endpoint — where logs are sent |
| `scrape_configs.job_name: docker` | Defines a scrape job for Docker logs |
| `__path__` | Glob pattern matching all Docker container JSON log files on the host |
| `pipeline_stages: docker: {}` | Parses Docker's JSON log format and extracts timestamp, stream (stdout/stderr), and the log message |

### Add Promtail Service to Docker Compose
Open `docker-compose.yml` and add:
```yaml
  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    volumes:
      - ./promtail/promtail-config.yml:/etc/promtail/promtail-config.yml
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
    command: -config.file=/etc/promtail/promtail-config.yml
    restart: unless-stopped
```

### Volume mounts explained:
| Mount | Purpose |
|---|---|
| `/var/lib/docker/containers` | Where Docker writes container log files - mounted read-only |
| `/var/run/docker.sock` | Lets Promtail discover container metadata (names, labels) for enrichment |
 
Without both mounts, Promtail would have no log files to read and no way to label them with container names.

### Restart the Stack
Bring everything up:
```bash
docker compose up -d
```
This starts Promtail alongside Loki, Grafana, and Prometheus.

### Generate Some Logs
Hit our **notes-app** to produce log entries:
```bash
for i in $(seq 1 20); do curl -s http://localhost:8000 > /dev/null; done
```
This simulates traffic, so Promtail has logs to ship into Loki.

### Verify Promtail Targets
Open Promtail’s status page:
```Code
http://localhost:9080/targets
```
We should see the `docker` job listed with discovered targets.

> If the targets page is empty, check that the volume mounts are correct and that Docker log files exist at `/var/lib/docker/containers/`.

## 4. Add Loki as a Grafana Datasource
We can add it **manually through the UI** or **auto-provision it with YAML**.

### Locate Grafana Provisioning Directory
Grafana supports auto-provisioning datasources via YAML. We already have this directory from Day 74:
```Code
grafana/provisioning/datasources/datasources.yml
```
If it doesn’t exist, create the path:
```bash
mkdir -p grafana/provisioning/datasources
nano grafana/provisioning/datasources/datasources.yml
```

### Update `datasources.yml`
Update `grafana/provisioning/datasources/datasources.yml`:
```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: false
```
What this does:
- **Prometheus datasource** → already set up, remains default.
- **Loki datasource** → added with URL `http://loki:3100`.
- **access: proxy** → Grafana proxies requests to Loki.
- **editable: false** → prevents accidental changes in the UI.

### Restart Grafana
Apply the new datasource configuration:
```bash
docker compose restart grafana
```
This reloads Grafana with the updated provisioning file.

### Verify Datasources
- Log in to Grafana (`http://localhost:3000`) →
- Go to **Connections > Data Sources**.
- You should now see **two datasources**:
  - Prometheus
  - Loki

### Alternative Manual Setup (UI)
- Go to **Connections > Data Sources > Add data source**.
- Select **Loki**.
- Set URL: `http://loki:3100`.
- Click **Save & Test**.

Either way, we should now have two datasources in Grafana: Prometheus and Loki.

## 5. Query Logs with LogQL
LogQL is Loki's query language - structurally similar to PromQL but designed for log streams. All queries run from **Explore** in Grafana (compass icon on the left sidebar → select **Loki** as the datasource).

### Open Grafana Explore
- Log in to Grafana (`http://localhost:3000`).
- On the left sidebar, click the **compass icon → Explore**.
- In the datasource dropdown, select **Loki**.

Now we’re ready to run LogQL queries.

### Run Basic Stream Selector
Start simple:
```logql
{job="docker"}
```
- This shows all logs scraped by Promtail from Docker containers.
- Think of it as the “up” query in Prometheus - a basic sanity check.

### Filter by Container Name
If we want logs from a specific container (say Prometheus):
```logql
{container_name="prometheus"}
```
This narrows down logs to just that container.

### Keyword Search
Find logs containing the word `error`:
```logql
{job="docker"} |= "error"
```
- `|=` means “line contains”. `|~` means "line matches regex".
- Case-sensitive by default. For case-insensitive, use regex:
  ```logql
  {job="docker"} |~ "(?i)error"
  ```

### Negative Filter
Exclude noisy health checks:
```logql
{job="docker"} != "health"
```
This removes log lines containing “health”.

### Regex Filter
Find HTTP 4xx or 5xx errors:
```logql
{job="docker"} |~ "status=[45]\\d{2}"
```
This matches status codes like 404, 500, etc.

### Log Metrics Queries
LogQL can turn logs into metrics:
- **Count logs over time**:
  ```logql
  count_over_time({job="docker"}[5m])
  ```
- **Rate of logs per second**:
  ```logql
  rate({job="docker"}[5m])
  ```
- **Top containers by log volume**:
  ```logql
  topk(5, sum by (container_name) (rate({job="docker"}[5m])))
  ```
### Write a LogQL query that finds all error logs from the notes-app container in the last 1 hour. Then write another query that counts how many error lines per minute.
**Ans.**
- **Find all error logs from notes-app in the last 1 hour**:
  ```logql
  {container_name="notes-app"} |= "error" | range(1h)
  ```
  (In Grafana Explore, you can set the time range to “Last 1 hour” instead of adding `| range(1h)`.)
- **Count how many error lines per minute**:
  ```logql
  count_over_time({container_name="notes-app"} |= "error" [1m])
  ```
This produces a graph showing error frequency over time - useful for spotting when errors started and whether they're increasing.

## 6. Correlate Metrics and Logs in Grafana
The real value of a unified observability stack is correlation - seeing metrics and logs together in the same view, at the same time.

### Add a Logs Panel to Your Dashboard
- Open the dashboard you built on **Day 74** (with CPU, memory, container metrics).
- Click **Add Panel → Add a new panel**.
- Select **Loki** as the datasource.
- Enter query:
  ```logql
  {job="docker"}
  ```
- Set **Visualization** to **Logs**.
- Title the panel: **Container Logs**.
- Save the dashboard.

Now your dashboard shows both metrics (Prometheus) and logs (Loki).

### Use Explore Split View
- Go to **Explore** (compass icon).
- Click the **Split button** (two panels side by side).
- Left panel → Select **Prometheus** datasource. Run:
  ```promql
  rate(container_cpu_usage_seconds_total{name="notes-app"}[5m])
  ```
- Right panel → Select **Loki** datasource. Run:
  ```logql
  {container_name="notes-app"}
  ```

Now we’ll see CPU usage spikes on the left and corresponding logs on the right.

### Time Synchronisation
Click any spike or anomaly in the metrics graph - both panels jump to that exact time range simultaneously. We can immediately inspect what the application was logging at the moment the anomaly occurred.
 
This is the core incident response workflow: **metrics surface the anomaly, logs explain it.**

### How does having metrics and logs in the same tool (Grafana) help during incident response compared to checking separate systems?
**Ans.** Using separate systems for metrics and logs during an incident means context-switching — pulling up Prometheus, noting the timestamp, switching to a log tool, filtering to that time range, cross-referencing container names. Every switch costs time and introduces the risk of misalignment.

Having metrics and logs in the same tool (Grafana) accelerates incident response. Instead of switching between Prometheus and a separate log system, you correlate anomalies and root causes in one place. This reduces context-switching, speeds up diagnosis, and helps teams respond faster.

## Complete Stack — Service Summary
After Day 75, your `docker compose ps` should show all six services running:
| Container | Port | Role |
|---|---|---|
| `prometheus` | 9090 | Metrics storage and PromQL engine |
| `node-exporter` | 9100 | Host-level metrics (CPU, memory, disk, network) |
| `cadvisor` | 8080 | Container-level metrics |
| `grafana` | 3000 | Dashboards, alerting, and unified explore UI |
| `loki` | 3100 | Log storage and LogQL engine |
| `promtail` | 9080 | Log collection agent — tails Docker log files |