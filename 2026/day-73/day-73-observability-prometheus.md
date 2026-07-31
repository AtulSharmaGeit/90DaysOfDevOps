# Introduction to Observability and Prometheus
> Infrastructure is built. Servers are configured. Containers are running. Now - how do we know it's all healthy? And when something breaks at 3 AM, how do we find out why?
 
That's the problem observability solves. This guide covers the three pillars of observability - metrics, logs, and traces - and walks through setting up Prometheus, the most widely used metrics collection tool in the DevOps ecosystem.
 
## Table of Contents
1. [Understanding Observability](#1-understand-observability)
2. [Set Up Prometheus with Docker](#2-set-up-prometheus-with-docker)
3. [Core Prometheus Concepts](#3-understand-prometheus-concepts)
4. [PromQL Basics](#4-learn-promql-basics)
5. [Add a Sample Application as a Scrape Target](#5-add-a-sample-application-as-a-scrape-target)
6. [Data Retention and Storage](#6-explore-data-retention-and-storage)

## 1. Understand Observability
Research and write short notes on:

### What is observability? How is it different from traditional monitoring?
**Ans.** These terms are often used interchangeably, but they answer different questions:
| | Monitoring | Observability |
|---|---|---|
| **Question** | *When* is something wrong? | *Why* is something wrong? |
| **Approach** | Alerts and thresholds | Exploration, querying, correlation |
| **Example** | "Error rate exceeded 5%" | "The payment service timed out calling the DB" |

### The Three Pillars of Observability:
**Metrics** - Numerical measurements over time: CPU usage, request count, error rate.<br>
**Tools**: Prometheus, Datadog, CloudWatch
 
**Logs** - Timestamped text records of events: application output, error messages, stack traces.<br>
**Tools**: Loki, ELK Stack, Fluentd
 
**Traces** - The journey of a single request across multiple services.<br>
**Tools**: OpenTelemetry, Jaeger, Zipkin

### Why do DevOps engineers need all three?
**Ans.** Each pillar answers a different part of the same incident:
- **Metrics** tell you *what* is broken → high error rate on `/api/users`
- **Logs** tell you *why* it broke → stack trace showing a database timeout
- **Traces** tell you *where* it broke → the payment service call took 12 seconds


### The Architecture We're Building
 Over the next five days, this is the full observability stack:
```
[Your App]  ──── metrics ──► [Prometheus]       ──► [Grafana Dashboards]
[Your App]  ──── logs    ──► [Promtail]         ──► [Loki] ──► [Grafana]
[Your App]  ──── traces  ──► [OTEL Collector]   ──► [Grafana / Debug]
[Host]      ──── metrics ──► [Node Exporter]    ──► [Prometheus]
[Docker]    ──── metrics ──► [cAdvisor]         ──► [Prometheus]
```

## 2. Set Up Prometheus with Docker
Create a project directory for this entire observability block - we will keep adding to it over the next 5 days.

### Create Project Directory
We’ll keep all observability files in one place for the next 5 days.
```bash
mkdir observability-stack && cd observability-stack
```

### Create Prometheus Configuration
Prometheus needs a config file (`prometheus.yml`) to know what endpoints to scrape.
`prometheus.yml`
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
```
| Field | Purpose |
|---|---|
| `scrape_interval` | How often Prometheus pulls metrics from targets (15s) |
| `evaluation_interval` | How often Prometheus evaluates alerting rules (15s) |
| `job_name` | Logical name for a group of targets |
| `targets` | Endpoints Prometheus scrapes (here, itself at `localhost:9090`) |

### Create Docker Compose File
Now let’s define how Prometheus runs inside Docker (`docker-compose.yml`).
```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    restart: unless-stopped

volumes:
  prometheus_data:
```
- **image** → official Prometheus Docker image.
- **ports** → maps container port 9090 to host port 9090.
- **volumes** → mounts config + persistent data storage.
- **restart: unless-stopped** → ensures Prometheus auto-restarts if it crashes.

The named volume `prometheus_data` persists metric history across container restarts. Without it, all historical data is lost whenever the container is recreated.

### Start Prometheus
```bash
docker compose up -d
```
- `-d` → detached mode (runs in background).
- Docker will pull the Prometheus image if not already present.

Verify the container is running:
```bash
docker ps
# Expected: container named 'prometheus' with port 9090 mapped
```

### Access Prometheus UI
Open our browser:
```Code
http://localhost:9090
```
We should see the **Prometheus web UI**.
- Navigate to **Status > Targets**.
- We should see **one target (prometheus)**.
- State should be **UP** (green).

## 3. Understand Prometheus Concepts
### How Prometheus Works
Prometheus uses a **pull-based model** - it reaches out to scrape targets at regular intervals, rather than waiting for targets to push data to it. Each scraped endpoint is called a **scrape target**.
 
### Metric Types
| Type | Behaviour | Example |
|---|---|---|
| **Counter** | Only goes up (total requests served, total errors) | `http_requests_total`, `errors_total` |
| **Gauge** | Increases and decreases (current CPU usage, memory in use, active connections) | `memory_usage_bytes`, `active_connections` |
| **Histogram** | Distribution of values in configurable buckets | Request duration bucketed as <100ms, <500ms, <1s |
| **Summary** | Similar to histogram, but percentiles calculated client-side | p50, p95, p99 latency |

### Labels and Time Series 
**Labels** are key-value pairs that add dimensions to a metric:
```
http_requests_total{method="GET", status="200", handler="/api/users"}
```
A **time series** is a unique combination of a metric name and a set of labels. Labels let you slice and filter data without creating separate metrics.

### Open Prometheus Graph UI
- Go to: http://localhost:9090/graph
- This is the query interface where you can run PromQL queries and visualize results.

We should see a text box for queries and options to view results as a graph or table.

### Run Queries in Prometheus
Run these queries one by one in the Graph UI:

```
# How many metrics is Prometheus collecting about itself?
count({__name__=~".+"})

# How much memory is Prometheus using?
process_resident_memory_bytes

# Total HTTP requests to the Prometheus server
prometheus_http_requests_total

# Break it down by handler
prometheus_http_requests_total{handler="/api/v1/query"}
```
We should see numeric results in the **Console tab** and graphs in the **Graph tab**.

### What is the difference between a counter and a gauge? Give one real-world example of each.
**Counter** → only increases.
- Example: `http_requests_total` → counts total requests served.
- Real-world analogy: Odometer in a car (always goes up).

**Gauge** → goes up and down.
- Example: `process_resident_memory_bytes` → memory usage fluctuates.
- Real-world analogy: Speedometer in a car (can increase or decrease).

## 4. Learn PromQL Basics
PromQL (Prometheus Query Language) is how you interrogate your metrics data. All queries run from `http://localhost:9090/graph`.

### Open the Graph UI
- Go to http://localhost:9090/graph.
- This is where we’ll type PromQL queries and see results as **Console output** or **Graphs**.

We should see a text box at the top and options for Graph and Console.

### Run Basic Queries
#### 1. Instant Vector
Returns the current value for each matching time series:
```promql
up
```
- Returns `1` if the target is healthy, `0` if down. You'll see one result per scrape target.
- Our Prometheus target should show `1`.

#### 2. Range Vector
Returns all values collected over a time window:
 ```promql
prometheus_http_requests_total[5m]
```
- Useful for inspecting raw samples and feeding into functions like `rate()`.
- We should see multiple values plotted over time.

#### 3. Rate Function
Converts a monotonically increasing counter into a per-second rate:
```promql
rate(prometheus_http_requests_total[5m])
```
- Because counters only go up, raw counter values are rarely useful on their own. `rate()` makes them meaningful by showing how fast the counter is increasing.
- We’ll see a smooth line showing requests per second.

#### 4. Aggregation
```promql
sum(rate(prometheus_http_requests_total[5m]))
```
- Sum across all label combinations to get a single aggregate value.
- Useful for overall traffic.

We’ll see a single value representing total request rate.

#### 5. Label Filtering
```promql
prometheus_http_requests_total{code="200"}
prometheus_http_requests_total{code!="200"}
```
- First query → only 200 OK requests.
- Second query → everything except 200.

We’ll see filtered results by HTTP status code.

#### 6. Arithmetic
```promql
process_resident_memory_bytes / 1024 / 1024
```
- Converts bytes → MB.
- Handy for readability.

We’ll see memory usage in MB.

#### 7. Top-K
```promql
topk(5, prometheus_http_requests_total)
```
- Shows top 5 metrics by value.
- Great for spotting heavy hitters.

We’ll see the top 5 request handlers.

### Write a PromQL query that shows the per-second rate of non-200 HTTP requests to Prometheus over the last 5 minutes.
```promql
rate(prometheus_http_requests_total{code!="200"}[5m])
```
This tells us how quickly errors (non-200s) are accumulating - a key signal for alerting.

## 5. Add a Sample Application as a Scrape Target
Prometheus becomes useful when it's monitoring something beyond itself. Add a sample application that exposes Prometheus-compatible metrics.

### Update `docker-compose.yml`
Add a **sample app** (`notes-app`) alongside Prometheus that exposes Prometheus metrics:
```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    restart: unless-stopped

  notes-app:
    image: trainwithshubham/notes-app:latest
    container_name: notes-app
    ports:
      - "8000:8000"
    restart: unless-stopped

volumes:
  prometheus_data:
```

### Update `prometheus.yml`
Tell Prometheus to scrape the new app:
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "notes-app"
    static_configs:
      - targets: ["notes-app:8000"]
```
Notice that the `notes-app` target uses the **container name** (`notes-app:8000`) rather than `localhost`. Within Docker's network, containers resolve each other by name.

### Restart the Stack
Apply changes by restarting:
```bash
docker compose up -d
```
Run `docker ps` → we should see **prometheus** and **notes-app** containers running.

### Verify in Prometheus UI
- Open http://localhost:9090 **→ Status > Targets**.
- You should now see **two targets**:
    - `prometheus` (scraping itself).
    - `notes-app` (scraping the sample app).

- Both should show **UP**. Both targets are green and healthy.

### Generate Traffic
Send requests to the sample app so Prometheus has data to scrape:
```bash
curl http://localhost:8000
curl http://localhost:8000
curl http://localhost:8000
```
Prometheus should now collect metrics from `notes-app`.

> **Note:** Not all applications expose Prometheus metrics natively. In later sessions you'll learn how **Node Exporter**, **cAdvisor**, and the **OpenTelemetry Collector** act as metric exporters for systems that don't have built-in Prometheus support.

## 6. Explore Data Retention and Storage
Understand how Prometheus stores data:
### Check Disk Usage
Prometheus stores metrics in its **local time‑series database (TSDB)** inside the container at `/prometheus`.

Run:
```bash
docker exec prometheus du -sh /prometheus
```
- `du -sh` → shows disk usage in human‑readable format.
- This tells you how much space Prometheus is currently using for its TSDB.

We should see output like `50M /prometheus`.

### Default Retention
By default, Prometheus retains data for **15 days**, after which older blocks are automatically deleted. This prevents unbounded disk growth but means historical analysis is limited to a two-week window.

### Configure Custom Retention
We can override defaults in `docker-compose.yml` under the Prometheus service:
```yaml
command:
  - '--config.file=/etc/prometheus/prometheus.yml'
  - '--storage.tsdb.retention.time=30d'
  - '--storage.tsdb.retention.size=1GB'
```
- `--storage.tsdb.retention.time=30d` → keep data for 30 days.
- `--storage.tsdb.retention.size=1GB` → cap TSDB size at 1GB regardless of time.
- You can use either **time** or **size**, or both.

### Check TSDB Status in UI
- Open http://localhost:9090 → **Status > TSDB Status**.
- This page shows:
    - Current block size.
    - Number of series.
    - Retention settings.
    - Disk usage.

We can confirm Prometheus is applying your retention settings.

### What happens when retention is exceeded?
- Prometheus automatically deletes the oldest data blocks.
- Only recent data (within retention window) is available for queries.
- This prevents uncontrolled disk growth.

### Why is a volume mount important for Prometheus data?
- Without a volume, TSDB data lives inside the container filesystem.
- If the container restarts or is removed, **all metrics history is lost**.
- With a mounted volume (`prometheus_data:/prometheus`), data persists across restarts.

The `prometheus_data:/prometheus` volume mount in `docker-compose.yml` is what makes Prometheus production-safe - without it, every restart is a blank slate.