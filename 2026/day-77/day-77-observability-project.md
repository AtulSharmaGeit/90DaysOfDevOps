# Observability Project: Full Stack with Docker Compose
> Four days of building — Prometheus, Node Exporter, cAdvisor, Grafana, Loki, Promtail, and the OpenTelemetry Collector — all come together here. One command spins up the complete 8-service stack. The goal today is end-to-end validation, a unified dashboard, and documentation we could hand off to a teammate.
 
## Table of Contents
1. [Clone and Launch the Reference Stack](#1-clone-and-launch-the-reference-stack)
2. [Validate the Metrics Pipeline](#2-validate-the-metrics-pipeline)
3. [Validate the Logs Pipeline](#3-validate-the-logs-pipeline)
4. [Validate the Traces Pipeline](#4-validate-the-traces-pipeline)
5. [Build a Unified Production Overview Dashboard](#5-build-a-unified-production-overview-dashboard)
6. [Compare and Document](#6-compare-your-stack-with-the-reference-and-document)

## 1. Clone and Launch the Reference Stack
### Clone the Repository that contains the Complete Observability Setup:
```bash
git clone https://github.com/LondheShubham153/observability-for-devops.git
cd observability-for-devops
```

### Examine the project structure:
```bash
tree -I 'node_modules|build|staticfiles|__pycache__'
```

```
observability-for-devops/
├── docker-compose.yml                      # All 8 services orchestrated
├── prometheus.yml                          # Prometheus scrape configuration
├── alert-rules.yml                         # Prometheus alerting rules
├── grafana/
│   └── provisioning/
│       ├── datasources/
│       │   └── datasources.yml             # Auto-provisioned: Prometheus + Loki
│       └── dashboards/
│           └── dashboards.yml              # Dashboard provisioning config
├── loki/
│   └── loki-config.yml                     # Loki storage and schema config
├── promtail/
│   └── promtail-config.yml                 # Docker log collection config
├── otel-collector/
│   └── otel-collector-config.yml           # OTLP receivers, processors, exporters
└── notes-app/                              # Sample Django + React application
```

### Launch the entire stack
Start all containers in detached mode.
```bash
docker compose up -d
```

Wait for all containers to start:
```bash
docker compose ps
```

### Verify Service Health
All 8 services should show as running:
| Service | Port | Verification |
|---|---|---|
| Prometheus | 9090 | `http://localhost:9090` |
| Node Exporter | 9100 | `curl http://localhost:9100/metrics \| head -5` |
| cAdvisor | 8080 | `http://localhost:8080` |
| Grafana | 3000 | `http://localhost:3000` (admin / admin) |
| Loki | 3100 | `curl http://localhost:3100/ready` → `ready` |
| Promtail | 9080 | `curl http://localhost:9080/targets` |
| OTEL Collector | 4317 / 4318 | `docker logs otel-collector` |
| Notes App | 8000 | `http://localhost:8000` |

## 2. Validate the Metrics Pipeline
Confirm Prometheus is scraping all targets:

### Open Prometheus Targets
- In your browser, go to: http://localhost:9090/targets
- You should see a list of scrape jobs. Confirm that **all 4 jobs are UP**:
    - **prometheus** → self-monitoring
    - **node-exporter** → host metrics
    - **cadvisor** → container metrics
    - **otel-collector** → OTLP metrics

All jobs show **green “UP”** status.

### Run Validation Queries
Go to http://localhost:9090/graph → paste each query → hit **Execute** → check results.

- **All targets healthy**
    ```promql
    up
    ```
    - Should return `1` for each target.

- **Host CPU usage**
    ```promql
    100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
    ```
    - Gauge of CPU usage percentage.

- **Memory usage**
    ```promql
    (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
    ```
    - Percentage of memory used.

- **Container CPU per container**
    ```promql
    rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100
    ```
    - Time series showing CPU usage per container.

- **Top 3 memory-hungry containers**
    ```promql
    topk(3, container_memory_usage_bytes{name!=""})
    ```
    - Bar chart of top 3 containers by memory usage.

Each query returns meaningful values (not empty). If you see “No data,” check that containers are running and traffic is being generated.

### Compare the `prometheus.yml` from the reference repo with the one you built over days 73-76. Note the scrape jobs and intervals.

## 3. Validate the Logs Pipeline
### Generate Application Logs
Promtail only ships logs if containers are producing them. Let’s hit the Notes App and its API to generate traffic:
```bash
for i in $(seq 1 50); do
  curl -s http://localhost:8000 > /dev/null
  curl -s http://localhost:8000/api/ > /dev/null
done
```
We should see HTTP requests in the Notes App container logs
```bash
docker logs notes-app | tail -20
```

### Explore Logs in Grafana
- Open Grafana → http://localhost:3000 (login: `admin/admin`).
- Click **Explore** (compass icon).
- Select **Loki** as the datasource.
- Run these LogQL queries one by one:
    - **All container logs**
        ```logql
        {job="docker"}
        ```

    - **Only notes-app logs**
        ```logql
        {container_name="notes-app"}
        ```

    - **Errors across all containers**
        ```logql
        {job="docker"} |= "error"
        ```

    - **HTTP request logs from the app**
        ```logql
        {container_name="notes-app"} |= "GET"
        ```

    - **Rate of log lines per container**
        ```logql
        sum by (container_name) (rate({job="docker"}[5m]))
        ```

We should see log lines streaming in Grafana. If we see “No data,” check that you generated traffic first and adjust the time range (Last 30m).

### Verify Promtail Targets
Promtail is the agent that tails Docker log files and ships them to Loki. Let’s confirm what it’s watching:
```bash
curl -s http://localhost:9080/targets | head -30
```

We’ll see JSON output listing log file paths under `/var/lib/docker/containers/...` and labels like `container_name`.

### Compare `promtail/promtail-config.yml` from the reference repo with yours from Day 75.

## 4. Validate the Traces Pipeline

### Send a Test OTLP Traces to the Collector
We’ll simulate a two-span trace (HTTP request + DB query) using curl:
```bash
curl -X POST http://localhost:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{
    "resourceSpans": [{
      "resource": {
        "attributes": [{
          "key": "service.name",
          "value": { "stringValue": "notes-app" }
        }]
      },
      "scopeSpans": [{
        "spans": [{
          "traceId": "aaaabbbbccccdddd1111222233334444",
          "spanId": "1111222233334444",
          "name": "GET /api/notes",
          "kind": 2,
          "startTimeUnixNano": "1700000000000000000",
          "endTimeUnixNano": "1700000000150000000",
          "attributes": [{
            "key": "http.method",
            "value": { "stringValue": "GET" }
          },
          {
            "key": "http.route",
            "value": { "stringValue": "/api/notes" }
          },
          {
            "key": "http.status_code",
            "value": { "intValue": "200" }
          }],
          "status": { "code": 1 }
        },
        {
          "traceId": "aaaabbbbccccdddd1111222233334444",
          "spanId": "5555666677778888",
          "parentSpanId": "1111222233334444",
          "name": "SELECT notes FROM database",
          "kind": 3,
          "startTimeUnixNano": "1700000000020000000",
          "endTimeUnixNano": "1700000000120000000",
          "attributes": [{
            "key": "db.system",
            "value": { "stringValue": "sqlite" }
          },
          {
            "key": "db.statement",
            "value": { "stringValue": "SELECT * FROM notes" }
          }]
        }]
      }]
    }]
  }'
```
This POST request sends a trace with two spans to the OTEL Collector.

### Check Collector Logs
Now confirm the collector received and processed the spans:
```bash
docker logs otel-collector 2>&1 | grep -A 20 "GET /api/notes"
```
We should see both spans printed in the logs:
- The **HTTP span** (`GET /api/notes`)
- The **DB span** (`SELECT * FROM notes`)

Look for:
- Attributes (`http.method`, `http.route`, `db.system`, `db.statement`)
- Parent-child relationship (`parentSpanId`)
- Timing data (`startTimeUnixNano`, `endTimeUnixNano`)

### Compare `otel-collector/otel-collector-config.yml` from the reference repo with yours from Day 76.

## 5. Build a Unified "Production Overview" Dashboard
A single dashboard that gives a complete, at-a-glance picture of the entire stack - host health, container workloads, application logs, and service-level telemetry.

### Open Grafana
- Go to http://localhost:3000 in our browser.
- Login with `admin/admin`.
- From the left menu, click **Dashboards → New → New Dashboard**.
- Click **Add a new panel**.

### Add Row 1 - System Health (Node Exporter + Prometheus)
For each panel, set **Datasource = Prometheus**.
| Panel | Type | Query |
|-------|------|-------|
| CPU Usage | Gauge | `100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)` |
| Memory Usage | Gauge | `(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100` |
| Disk Usage | Gauge | `(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100` |
| Targets Up | Stat | `sum(up)` / `count(up)` |

We now have a quick health overview of host + Prometheus targets.

### Add Row 2 - Container Metrics (cAdvisor)
Datasource = **Prometheus** (cAdvisor metrics).
| Panel | Visualisation | Query | Legend |
|---|---|---|---|
| Container CPU | Time series | `rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100` | `{{name}}` |
| Container Memory | Bar chart | `container_memory_usage_bytes{name!=""} / 1024 / 1024` | `{{name}}` |
| Container Count | Stat | `count(container_last_seen{name!=""})` | — |

We can now see per-container CPU, memory, and count.

### Add Row 3 - Application Logs (Loki)
Datasource = **Loki**.
| Panel | Type | Query |
|-------|------|-------|
| App Logs | Logs | `{container_name="notes-app"}` |
| Error Rate | Time series | `sum(rate({job="docker"} \|= "error" [5m]))` |
| Log Volume | Time series | `sum by (container_name) (rate({job="docker"}[5m]))` |

We can now correlate logs with metrics directly in Grafana.

### Add Row 4 - Service Overview
Datasource = **Prometheus**.
| Panel | Type | Query |
|-------|------|-------|
| Prometheus Scrape Duration | Time series | `prometheus_target_interval_length_seconds{quantile="0.99"}` |
| OTEL Metrics Received | Stat | `otelcol_receiver_accepted_metric_points` (if available) |

This row shows service-level health.

### Save & Configure Dashboard
- Click **Save Dashboard**.
- Name it: **Production Overview - Observability Stack**.
- Set **Time range = Last 30 minutes**.
- Enable **Auto-refresh = every 10s**.

We now have a single pane of glass for system health, container metrics, logs, and service overview.

## 6. Compare Your Stack with the Reference and Document
### Config Comparison
Review each configuration file side by side - the version we built across Days 73–76 versus the reference repo:
| File | Your Location | Reference Location | What to Compare |
|---|---|---|---|
| `prometheus.yml` | Project root | Root directory | Scrape jobs, intervals, target labels |
| `loki-config.yml` | `loki/` | `loki/` | Storage backend, schema version, retention |
| `promtail-config.yml` | `promtail/` | `promtail/` | Scrape configs, pipeline stages, label extraction |
| `otel-collector-config.yml` | `otel-collector/` | `otel-collector/` | Receivers, processors, exporters, pipelines |
| `datasources.yml` | `grafana/provisioning/` | `grafana/provisioning/` | Datasource types, URLs, default settings |
| `docker-compose.yml` | Project root | Root directory | All 8 services, volumes, port mappings |
 
### Week in Review
| Day | What You Built |
|---|---|
| 73 | Prometheus fundamentals - scraping, PromQL, data retention |
| 74 | Node Exporter (host metrics), cAdvisor (container metrics), Grafana dashboards |
| 75 | Loki (log storage), Promtail (log collection), LogQL, metric-log correlation |
| 76 | OTEL Collector (traces), Prometheus alerting rules, Grafana alerts |
| 77 | Full stack integration, end-to-end validation, unified dashboard |
 
### Production Enhancements
What we'd add before running this in production:
| Enhancement | Why |
|---|---|
| **Alertmanager** | Routes Prometheus alerts to Slack, PagerDuty, email with deduplication and silencing |
| **Grafana Tempo** | Persistent trace storage and visual trace exploration to replace the debug exporter |
| **HTTPS / TLS** | Encrypts traffic to all endpoints - Grafana, Prometheus, Loki, OTEL Collector |
| **Authentication** | Locks down Prometheus and Loki, which have no auth by default |
| **Log retention policies** | Prevents unbounded Loki disk growth in long-running environments |
| **HA replicas** | Multiple Prometheus and Loki instances with shared object storage for reliability |
 
### How does this stack compare to managed solutions like Datadog, New Relic, or AWS CloudWatch?
| Dimension | This Stack (self-hosted) | Managed (Datadog, New Relic, CloudWatch) |
|---|---|---|
| **Cost** | Infrastructure cost only | Per-host or per-metric pricing - can be significant at scale |
| **Control** | Full - tune retention, cardinality, pipelines | Limited to vendor configuration options |
| **Operational overhead** | You maintain upgrades, storage, HA | Vendor-managed |
| **Vendor lock-in** | None - all components are open source | High - proprietary query languages and APIs |
| **Setup time** | Hours to days | Minutes |
| **Best for** | Teams with DevOps capacity who need control and cost efficiency | Teams prioritising speed and minimal operational burden |
 
### Clean Up
```bash
# Remove all containers and named volumes (Prometheus data, Grafana data, Loki data)
docker compose down -v
```
> Only use the `-v` flag when you are finished exploring. It permanently deletes all stored metrics, logs, and dashboard configurations.