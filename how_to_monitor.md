# Monitoring with Prometheus, VictoriaMetrics, and Grafana

This guide explains how to monitor your Docker Swarm services, nodes, and databases using Prometheus (scraping metrics), VictoriaMetrics (long-term storage), and Grafana (visualization).

---

## What You Can Monitor Now

- **Backapp Service Metrics:**
  - HTTP request count (`http_requests_total`)
  - Custom metrics exposed on `/metrics` endpoint

- **Node Exporter Metrics (host metrics):**
  - CPU usage
  - Memory usage
  - Disk usage
  - Network statistics

- **MongoDB & Redis Metrics:**
  - MongoDB and Redis do not expose Prometheus metrics out-of-the-box here but can be monitored using exporters (optional to add later)

- **VictoriaMetrics:**
  - Long-term metrics storage and query engine compatible with PromQL

---

## Prometheus Query Examples

| Metric/Goal                | PromQL Query                                      | Description                       |
|---------------------------|-------------------------------------------------|---------------------------------|
| Total HTTP Requests       | `http_requests_total`                            | Total number of HTTP requests   |
| HTTP Request Rate (per sec)| `rate(http_requests_total[1m])`                  | Requests per second over 1 min  |
| CPU Usage (node)           | `100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)` | CPU usage % excluding idle      |
| Memory Usage (node)        | `(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100` | Memory usage %                  |
| Disk Usage (node)          | `(node_filesystem_size_bytes{mountpoint="/"} - node_filesystem_free_bytes{mountpoint="/"}) / node_filesystem_size_bytes{mountpoint="/"} * 100` | Disk usage % on root partition  |
| Network Receive Bytes      | `rate(node_network_receive_bytes_total[1m])`    | Network receive bytes per sec   |
| Network Transmit Bytes     | `rate(node_network_transmit_bytes_total[1m])`   | Network transmit bytes per sec  |

---

## Grafana Setup for Visualization

1. **Add Prometheus Data Source:**
   - URL: `http://prometheus:9090` (or your Prometheus URL)
   - Access: Server (default)

2. **Create Dashboards:**

   ### Example Panel Queries:

   - **HTTP Requests Over Time:**

     ```promql
     rate(http_requests_total[1m])
     ```

   - **CPU Usage:**

     ```promql
     100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
     ```

   - **Memory Usage:**

     ```promql
     (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100
     ```

   - **Disk Usage:**

     ```promql
     (node_filesystem_size_bytes{mountpoint="/"} - node_filesystem_free_bytes{mountpoint="/"}) / node_filesystem_size_bytes{mountpoint="/"} * 100
     ```

   - **Network Traffic Receive:**

     ```promql
     rate(node_network_receive_bytes_total[1m])
     ```

   - **Network Traffic Transmit:**

     ```promql
     rate(node_network_transmit_bytes_total[1m])
     ```

---

## VictoriaMetrics Usage

- Acts as long-term storage backend.
- Query it using the same PromQL queries as Prometheus.
- Grafana can be configured to use VictoriaMetrics as a datasource by setting URL to `http://victoriametrics:8428`.

---

## Summary

| Component         | What It Does                              | How to Access                          |
|-------------------|------------------------------------------|--------------------------------------|
| **Prometheus**    | Scrapes and queries real-time metrics    | `http://<host>:9090`                  |
| **VictoriaMetrics**| Stores metrics long-term and supports PromQL | `http://<host>:8428`               |
| **Grafana**       | Visualizes metrics dashboards             | `http://<host>:3000`                  |

---

## Tips

- Use Grafana dashboards to combine node and service metrics.
- Create alerts in Prometheus or Grafana based on key metrics.
- Extend by adding exporters for MongoDB and Redis if needed.

---

*Copy this entire file and use it as your Monitoring.md for quick reference on monitoring queries and setup.*

---------------------------------------------------------------------------------------------------------------------

# Monitoring Docker Swarm, Containers, MongoDB, and Redis with Prometheus, VictoriaMetrics, and Grafana

---

## 1. Overview

You can monitor:

- **Docker Swarm & Docker daemon metrics** (using `cAdvisor` and `node-exporter`)
- **Per-container metrics** (CPU, memory, network)
- **MongoDB metrics** (via MongoDB Exporter)
- **Redis metrics** (via Redis Exporter)
- **Store metrics long-term with VictoriaMetrics**
- **Visualize everything in Grafana**

---

## 2. How to Set Up Exporters & Collectors

### a. Docker Swarm & Docker Daemon / Containers

- Use **cAdvisor** to monitor containers (CPU, memory, disk IO, network).
- Use **node-exporter** to monitor host machine metrics (CPU, memory, disk, network).
- Prometheus scrapes both.

### b. MongoDB Monitoring

- Use [mongodb_exporter](https://github.com/percona/mongodb_exporter) to expose MongoDB metrics.

### c. Redis Monitoring

- Use [redis_exporter](https://github.com/oliver006/redis_exporter) to expose Redis metrics.

---

## 3. Example Docker Compose snippet for Monitoring Stack

```yaml
version: '3.8'
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"
    deploy:
      mode: global
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro

  mongodb-exporter:
    image: percona/mongodb_exporter:latest
    environment:
      - MONGODB_URI=mongodb://mongo_user:mongo_pass@mongo:27017
    ports:
      - "9216:9216"
    depends_on:
      - mongo

  redis-exporter:
    image: oliver006/redis_exporter:latest
    ports:
      - "9121:9121"
    environment:
      - REDIS_ADDR=redis:6379
    depends_on:
      - redis

  victoriametrics:
    image: victoriametrics/victoria-metrics:latest
    ports:
      - "8428:8428"

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana-storage:/var/lib/grafana
    depends_on:
      - prometheus
      - victoriametrics

volumes:
  grafana-storage:

## 4. Example `prometheus.yml` Configuration

```yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['prometheus:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'mongodb-exporter'
    static_configs:
      - targets: ['mongodb-exporter:9216']

  - job_name: 'redis-exporter'
    static_configs:
      - targets: ['redis-exporter:9121']

## 5. Prometheus Queries for Key Metrics
| Use Case                  | PromQL Query                                                            |
|---------------------------|------------------------------------------------------------------------|
| Container CPU usage        | rate(container_cpu_usage_seconds_total{image!=""}[1m])                |
| Container memory usage     | container_memory_usage_bytes{image!=""}                               |
| Node CPU usage            | 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) |
| Node memory usage         | (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100 |
| MongoDB connections       | mongodb_up (from mongodb_exporter)                                   |
| MongoDB operation counters| mongodb_op_counters_total{type="insert"}                            |
| Redis used memory         | redis_memory_used_bytes                                               |
| Redis connected clients   | redis_connected_clients                                               |

## 6. Visualizing Metrics in Grafana
- Add **Prometheus** and/or **VictoriaMetrics** as Data Sources in Grafana.
- Use ready-made dashboards for:
  - **Docker & cAdvisor**: Search “Docker” or “cAdvisor” dashboards on Grafana.com
  - **Node Exporter**: Import Node Exporter Full dashboard (ID: 1860)
  - **MongoDB**: Search “MongoDB Exporter” dashboards (e.g., ID: 2589)
  - **Redis**: Search “Redis” dashboards (e.g., ID: 763)


## 7. Summary & Tips
- Use **node-exporter** and **cAdvisor** for Docker daemon, Swarm nodes, and containers.
- Use **mongodb_exporter** and **redis_exporter** for database metrics.
- Store metrics long-term with **VictoriaMetrics**.
- Visualize & alert with **Grafana**.
- Extend by adding exporters for other services as needed.
- Secure your endpoints before production use.

## 8. Useful Link
- [cAdvisor GitHub](https://github.com/google/cadvisor)
- [node-exporter GitHub](https://github.com/prometheus/node_exporter)
- [mongodb_exporter GitHub](https://github.com/percona/mongodb_exporter)
- [redis_exporter GitHub](https://github.com/oliver006/redis_exporter)
- [VictoriaMetrics](https://github.com/VictoriaMetrics/VictoriaMetrics)
- [Grafana Dashboards](https://grafana.com/grafana/dashboards)
