# OpenTelemetry and Alerting
>We have metrics (Prometheus) and logs (Loki). This guide adds the third pillar - distributed traces - using OpenTelemetry, the industry-standard framework for telemetry collection. It then closes the loop with alerting, so our stack notifies us when something breaks instead of waiting for us to notice.

By the end, our observability stack covers all three pillars and actively fires alerts on real conditions.

## Table of Contents 
1. [Understanding OpenTelemetry](#1-understand-opentelemetry)
2. [Add the OpenTelemetry Collector](#2-add-the-opentelemetry-collector)
3. [Send Test Traces and Metrics](#3-send-test-traces-&-metrics-to-the-collector)
4. [Set Up Prometheus Alerting Rules](#4-set-up-prometheus-alerting-rules)
5. [Set Up Grafana Alerts](#5-set-up-grafana-alerts)
6. [Full Stack Architecture Review](#6-review-the-full-stack-architecture)

## 1. Understand OpenTelemetry
### What is OpenTelemetry (OTEL)?
**Ans.** OpenTelemetry is a vendor-neutral, open-source framework for generating, collecting, and exporting telemetry data - metrics, logs, and traces. It is not a storage backend; it collects data and ships it to backends like Prometheus, Loki, Jaeger, or Datadog.
 
The key benefit is standardisation: instrument our application once with the OTEL SDK, then route telemetry to any backend without changing application code.
 
### The OTEL Collector Pipeline
**Ans.** The OTEL Collector is a standalone service that sits between our applications and our backends. Every pipeline has three stages:
 
| Stage | Role | Examples |
|---|---|---|
| **Receivers** | Accept incoming telemetry | OTLP (gRPC/HTTP), Prometheus, Jaeger |
| **Processors** | Transform data in-flight | Batching, filtering, sampling, attribute enrichment |
| **Exporters** | Send data to backends | Prometheus, Jaeger, Tempo, debug console, Datadog |
 
### What is OTLP?
**Ans.** OTLP (OpenTelemetry Protocol) is the standard wire format for transmitting telemetry data between OTEL-instrumented applications and collectors or backends. It supports two transports:
 
| Transport | Port | Use Case |
|---|---|---|
| gRPC | 4317 | High-throughput, streaming - preferred for production |
| HTTP | 4318 | Simpler, firewall-friendly - easier for testing |
 
### What are Distributed Traces?
 **Ans.** A **trace** tracks a single request as it travels through multiple services. Each operation within the trace is a **span**.
```
User Request
└── API Gateway (span 1)         [0ms – 12ms]
    ├── Auth Service (span 2)    [2ms – 8ms]
    └── Database Query (span 3)  [9ms – 12ms]
```
Every span carries: a trace ID, a span ID, a parent span ID, start time, duration, and arbitrary key-value attributes. The parent span ID is what links spans into a tree, showing exactly where time was spent across service boundaries.

## 2. Add the OpenTelemetry Collector
### Create a Directory for Collector Config
```bash
mkdir -p otel-collector
cd otel-collector
```
This keeps our collector configuration organized in its own folder.

### Write the Collector Config File
`otel-collector/otel-collector-config.yml`:
```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:

exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"
  debug:
    verbosity: detailed

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug]
```

**What this config does:**
- **Receivers:** Accepts OTLP data via gRPC (4317) and HTTP (4318)
- **Processors:** Batches data before exporting (reduces overhead)
- **Exporters:**
  - Metrics go to a Prometheus-compatible endpoint on port 8889 (Prometheus scrapes this)
  - Traces and logs go to debug output (console) -- in production you would send these to Jaeger or Tempo

### Add Collector Service to Docker Compose
`docker-compose.yml`
```yaml
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    container_name: otel-collector
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8889:8889"   # Prometheus exporter
    volumes:
      - ./otel-collector/otel-collector-config.yml:/etc/otelcol-contrib/config.yaml
    restart: unless-stopped
```

### Add OTEL Collector as a Prometheus Scrape Target
`prometheus.yml`
```yaml
  - job_name: "otel-collector"
    static_configs:
      - targets: ["otel-collector:8889"]
```
This tells Prometheus to scrape metrics from the collector.

### Restart the Stack
```bash
docker compose up -d
```
This will start/restart all services including the new collector.

### Verify Collector is Running
Check logs:
```bash
docker logs otel-collector 2>&1 | tail -5
```
Check Prometheus targets:
- Open http://localhost:9090 → **Status → Targets**.
- We should see `otel-collector` listed as **UP**.

## 3. Send Test Traces & Metrics to the Collector
With the collector running, send telemetry directly via `curl` to verify the pipeline end-to-end.

### Send a Sample OTLP Trace
Run this `curl` command from our terminal:
```bash
curl -X POST http://localhost:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{
    "resourceSpans": [{
      "resource": {
        "attributes": [{
          "key": "service.name",
          "value": { "stringValue": "my-test-service" }
        }]
      },
      "scopeSpans": [{
        "spans": [{
          "traceId": "5b8efff798038103d269b633813fc60c",
          "spanId": "eee19b7ec3c1b174",
          "name": "test-span",
          "kind": 1,
          "startTimeUnixNano": "1544712660000000000",
          "endTimeUnixNano": "1544712661000000000",
          "attributes": [{
            "key": "http.method",
            "value": { "stringValue": "GET" }
          },
          {
            "key": "http.status_code",
            "value": { "intValue": "200" }
          }]
        }]
      }]
    }]
  }'
```
This sends a **trace** with one span (`test-span`) to the OTEL Collector via HTTP OTLP.

### Verify Trace in Collector Logs
Check the collector’s debug output:
```bash
docker logs otel-collector 2>&1 | grep -A 10 "test-span"
```
We should see details like `traceId`, `spanId`, `http.method=GET`, `http.status_code=200`.
This confirms the collector received and processed our trace.

>In a production setup, we would send these to a trace backend like Jaeger or Grafana Tempo for storage and visualization.

### Send a Sample OTLP Metric
Now push a test metric:
```bash
curl -X POST http://localhost:4318/v1/metrics \
  -H "Content-Type: application/json" \
  -d '{
    "resourceMetrics": [{
      "resource": {
        "attributes": [{
          "key": "service.name",
          "value": { "stringValue": "my-test-service" }
        }]
      },
      "scopeMetrics": [{
        "metrics": [{
          "name": "test_requests_total",
          "sum": {
            "dataPoints": [{
              "asInt": "42",
              "startTimeUnixNano": "1544712660000000000",
              "timeUnixNano": "1544712661000000000"
            }],
            "aggregationTemporality": 2,
            "isMonotonic": true
          }
        }]
      }]
    }]
  }'
```
This sends a **counter metric** (`test_requests_total = 42`) to the collector.

### Verify Metric in Prometheus
- Open Prometheus UI at http://localhost:9090 → Graph tab.
- Run the query:
    ```Code
    test_requests_total
    ```
- We should see the value `42`.

### The Full Journey of this Metric:
```
curl → OTEL Collector (OTLP receiver) → batch processor → Prometheus exporter (port 8889) → Prometheus scrape
```
> In production, traces would be routed to Jaeger or Grafana Tempo for storage and visualisation rather than the debug console.

## 4. Set Up Prometheus Alerting Rules
Alerts notify us when something is wrong. Prometheus evaluates alerting rules on a schedule and transitions alerts through three states: **Inactive → Pending → Firing**. The `for` duration prevents flapping by requiring a condition to hold for a minimum period before the alert fires.

### Create the Alert Rules File
Inside our project root, create a file named `alert-rules.yml`:
```yaml
groups:
  - name: system-alerts
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage detected"
          description: "CPU usage has been above 80% for more than 2 minutes. Current value: {{ $value }}%"

      - alert: HighMemoryUsage
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 85
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage detected"
          description: "Memory usage is above 85%. Current value: {{ $value }}%"

      - alert: ContainerDown
        expr: absent(container_last_seen{name="notes-app"})
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Container is down"
          description: "The notes-app container has not been seen for over 1 minute"

      - alert: TargetDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Scrape target is down"
          description: "{{ $labels.job }} target {{ $labels.instance }} is unreachable"

      - alert: HighDiskUsage
        expr: (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk space running low"
          description: "Root filesystem usage is above 90%. Current value: {{ $value }}%"
```
| Field | Purpose |
|---|---|
| `expr` | PromQL condition that must be true to trigger |
| `for` | Minimum duration the condition must hold before firing — prevents alert flapping |
| `labels.severity` | Used by notification routing to distinguish warnings from critical alerts |
| `annotations` | Human-readable context shown in alert notifications; supports `{{ $value }}` and `{{ $labels }}` templating |
### Update Prometheus Config to Load the Rules
`prometheus.yml`
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - /etc/prometheus/alert-rules.yml

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]

  - job_name: "cadvisor"
    static_configs:
      - targets: ["cadvisor:8080"]

  - job_name: "otel-collector"
    static_configs:
      - targets: ["otel-collector:8889"]
```

### Mount Rules in Docker Compose
Update our `docker-compose.yml` Prometheus service:
```yaml
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alert-rules.yml:/etc/prometheus/alert-rules.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    restart: unless-stopped
```

### Restart Prometheus
```bash
docker compose up -d prometheus
```

### Verify Rules in Prometheus UI
- Open http://localhost:9090.
- Go to **Status → Rules** → we should see all five alert rules listed.
- Go to **Alerts** → they should be in `inactive` state (green). If any condition is true, the alert moves to `pending`, then `firing` after the `for` duration.

### Test an Alert
Stop the notes-app container:
```bash
docker compose stop notes-app
```
Wait 1–2 minutes → check **Alerts** in Prometheus UI.
- The `ContainerDown` and `TargetDown` alerts should transition: **Inactive → Pending → Firing**.

Restart the container:
```bash
docker compose start notes-app
```
Alerts should return to **Inactive**.

## 5. Set Up Grafana Alerts
Grafana's alerting layer evaluates rules against your datasources and routes notifications to contact points - Slack, email, PagerDuty, webhooks, and more.

### Create a Contact Point
- Open Grafana at `http://localhost:3000`.
- Go to **Alerting → Contact points → Add contact point**.
- Fill in details:
    - **Name**: `DevOps Team`
    - **Integration**: Choose **Email** (or Slack webhook if available).
    - For email: enter your address (e.g., `devops-team@example.com`).
- Click **Save**.

This defines where alerts will be sent.

### Create an Alert Rule in Grafana
- Go to **Alerting → Alert rules → New alert rule**.
- Fill in:
    - **Name**: `High Container Memory`
    - **Query**:
        ```Code
        container_memory_usage_bytes{name="notes-app"} / 1024 / 1024
        ```
        (This converts bytes → MB).
    - **Condition**: IS ABOVE `100` (fires if container uses >100MB).
    - **Evaluation**: every `1m`, for `2m`.
    - **Label**: `severity = warning`.
    - **Contact point**: Link to `DevOps Team`.
- Click **Save**.

This rule continuously checks container memory usage and fires if it stays above 100MB for 2 minutes.

### Create a Notification Policy
- Go to **Alerting → Notification policies**.
- Set **Default contact point** = `DevOps Team`.
- Add a nested policy:
    - Match label `severity=critical`.
    - Route to a different contact point (or same one with different settings).

This lets us route warnings vs critical alerts differently.

### View Alert State
- Go to **Alerting → Alert rules**.
- We’ll see our rule in one of three states:
    - **Normal** (green) → condition not met.
    - **Pending** (yellow) → condition met but waiting for `for` duration.
    - **Firing** (red) → condition met and alert is active.

### What is the difference between Prometheus alerts and Grafana alerts? When would you use each?
| | Prometheus Alerts | Grafana Alerts |
|---|---|---|
| **Where evaluated** | At the Prometheus server | In Grafana, against any datasource |
| **Notification routing** | Requires Alertmanager | Built-in contact points (Slack, email, PagerDuty) |
| **Strength** | Deep infrastructure rules, close to the data | Visual, multi-datasource, team-friendly workflows |
| **Best for** | Infrastructure health thresholds | Actionable notifications routed to the right team |
 
Use Prometheus alerting rules for infrastructure-level conditions. Use Grafana alerts when you need notification routing, multi-datasource conditions, or team-facing workflows.

## 6. Review the Full Stack Architecture
The observability stack now covers all three pillars. Here is the complete data flow:
```
METRICS PIPELINE
─────────────────────────────────────────────────────────────
[Node Exporter]      ──scrape──► [Prometheus] ──► [Grafana]
[cAdvisor]           ──scrape──► [Prometheus] ──► [Grafana]
[OTEL Collector:8889]──scrape──► [Prometheus] ──► [Grafana]
                                 [Prometheus] ──► [Alert Rules → Notifications]
 
LOGS PIPELINE
─────────────────────────────────────────────────────────────
[Docker Containers] ──► [Promtail] ──► [Loki] ──► [Grafana]
 
TRACES PIPELINE
─────────────────────────────────────────────────────────────
[App / curl OTLP] ──► [OTEL Collector] ──► [Debug Console]
                                           (Production: Jaeger / Tempo)
```
 
### Complete Service Inventory
```bash
docker compose ps
# All 8 containers should show Up
```
 
| Container | Port(s) | Role |
|---|---|---|
| `prometheus` | 9090 | Metrics storage and PromQL engine |
| `node-exporter` | 9100 | Host-level metrics (CPU, memory, disk, network) |
| `cadvisor` | 8080 | Container-level metrics |
| `grafana` | 3000 | Dashboards, alerting, and unified Explore UI |
| `loki` | 3100 | Log storage and LogQL engine |
| `promtail` | 9080 | Log collection agent — tails Docker log files |
| `otel-collector` | 4317, 4318, 8889 | Telemetry collection — OTLP receiver, Prometheus exporter |
| `notes-app` | 8000 | Sample application |



