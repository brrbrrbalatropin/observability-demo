# observability-demo

![Java](https://img.shields.io/badge/Java-21-orange)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.1.0-brightgreen)
![Maven](https://img.shields.io/badge/Maven-build-blue)
![Prometheus](https://img.shields.io/badge/Prometheus-metrics-e6522c)
![Grafana](https://img.shields.io/badge/Grafana-dashboards-f46800)
![Loki](https://img.shields.io/badge/Loki-logs-yellow)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ed)

A hands-on **observability** demo for a Spring Boot application, built for the **Software
Architecture (ARSW)** course at Escuela Colombiana de Ingeniería Julio Garavito.

Its goal is to show, with a minimal but fully working example, the **three pillars of
observability** and how to wire them up with industry-standard tools:

| Pillar | Question it answers | Tool used here |
|--------|---------------------|----------------|
| **Metrics** | How many requests? How much latency? How many errors? | Prometheus (collects) + Micrometer (produces) |
| **Logs** | What exactly happened at this moment? | Loki (stores) + Promtail (collects) |
| **Visualization** | How do I see everything together in charts? | Grafana |

---

## Architecture

The Spring Boot application runs on your machine (port **8081**) and exposes its metrics. The
entire observability stack runs in **Docker** and consumes that data.

```
┌─────────────────────────┐
│  Spring Boot application │  (runs on your machine, port 8081)
│  observability-demo      │
│                          │
│  /orders  ← endpoints    │
│  /actuator/prometheus    │────┐  exposes metrics in Prometheus format
└─────────────────────────┘    │
                               │ scrape every 5s
        ┌──────────────────────▼──────────────────────┐
        │                  DOCKER                      │
        │                                              │
        │   Prometheus ──────► Grafana ◄────── Loki    │
        │   (:9090)            (:3000)        (:3100)  │
        │   metrics           dashboards      logs     │
        │                                       ▲      │
        │                                  Promtail    │
        │                              (collects logs) │
        └──────────────────────────────────────────────┘
```

> **Note:** The application does **not** run inside Docker. Prometheus reaches it through
> `host.docker.internal:8081`, which from within a container points to your host machine.

---

## Tech stack

- **Java 21** + **Spring Boot 4.1.0** + **Maven**
- **Spring Boot Actuator** — exposes health and metrics endpoints
- **Micrometer** (`micrometer-registry-prometheus`) — produces metrics in Prometheus format
- **Prometheus** — time-series database that collects the metrics
- **Grafana** — dashboards and visualization
- **Loki** + **Promtail** — log aggregation and collection
- **Docker Compose** — orchestrates the observability stack

---

## Project structure

```
observability-demo/
├── src/main/java/edu/eci/arsw/observability/
│   ├── ObservabilityDemoApplication.java     # Spring Boot entry point
│   └── controller/
│       ├── OrderController.java               # Order endpoints + custom metrics
│       └── GlobalExceptionHandler.java        # Global exception handling (returns 500)
├── src/main/resources/
│   └── application.yaml                        # Config: port, actuator, logging
├── prometheus/
│   └── prometheus.yml                          # Prometheus scraping config
├── loki/
│   └── loki-config.yml                         # Log storage config
├── promtail/
│   └── promtail-config.yml                     # Log collection config
├── docker-compose.yml                          # Stack: Prometheus, Grafana, Loki, Promtail
└── pom.xml                                      # Maven dependencies
```

---

## Prerequisites

- **JDK 21** installed
- **Docker Desktop** installed and running
- **Maven** (or use the included `./mvnw` wrapper)

---

## Running the project

### 1. Start the observability stack (Docker)

```bash
docker compose up -d
```

This brings up 4 containers:

| Service | Port | URL |
|---------|------|-----|
| Prometheus | 9090 | http://localhost:9090 |
| Grafana | 3000 | http://localhost:3000 (user: `admin`, password: `admin`) |
| Loki | 3100 | http://localhost:3100 |
| Promtail | — | (collects logs, no UI) |

### 2. Start the Spring Boot application

```bash
./mvnw spring-boot:run
```

The app becomes available at **http://localhost:8081**.

> Sanity check: http://localhost:8081/actuator/health should respond with `{"status":"UP"}`.

---

## Application endpoints

Everything lives under `/orders`:

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/orders` | Creates an order. Increments the `orders_created_total` metric. |
| `GET` | `/orders/{id}` | Fetches an order by id. |
| `GET` | `/orders/simulate-latency` | Simulates random latency (500–3000 ms). Useful to see latency in metrics. |
| `GET` | `/orders/simulate-error` | Throws a 500 error on purpose. Increments `orders_failed_total`. |

### Example requests

**Create an order:**
```bash
curl -X POST http://localhost:8081/orders \
  -H "Content-Type: application/json" \
  -d "{\"customerId\":\"CUS-01\",\"total\":120000}"
```

**Trigger an error (to inspect it later in metrics and logs):**
```bash
curl http://localhost:8081/orders/simulate-error
```

**Trigger latency:**
```bash
curl http://localhost:8081/orders/simulate-latency
```

---

## Viewing metrics

### Raw endpoint on the app

```
http://localhost:8081/actuator/prometheus
```

This shows **exactly** how each metric is named in Prometheus format. It's the first place to
look when a metric "doesn't show up".

### In Prometheus

1. Open **http://localhost:9090**
2. Type a metric name in the query box and click **Execute**
3. Use the **Table** tab to see the current value, or **Graph** to see it over time

> Prometheus starts with an **empty** screen: you won't see anything until you type a query and
> click Execute. This is normal.

**Useful queries (PromQL):**

```promql
# HTTP requests accumulated per endpoint
http_server_requests_seconds_count

# Total orders created successfully (custom metric)
orders_created_total

# Total failed orders (custom metric)
orders_failed_total

# Requests per second over the last minute
rate(http_server_requests_seconds_count[1m])

# Only the 500 errors
http_server_requests_seconds_count{status="500"}

# Average latency per endpoint
rate(http_server_requests_seconds_sum[1m]) / rate(http_server_requests_seconds_count[1m])
```

---

## Custom metrics

`OrderController.java` defines two custom counters with Micrometer:

```java
this.orderCreatedCounter = Counter.builder("orders.created")
        .description("Total orders created successfully")
        .register(meterRegistry);

this.orderFailedCounter = Counter.builder("orders.failed")
        .description("Total failed orders")
        .register(meterRegistry);
```

> **Important Micrometer convention:** names are written with **dots** and **without the
> `_total` suffix**. Micrometer converts `orders.created` into the Prometheus name
> `orders_created_total` (it replaces dots with underscores and automatically appends `_total`
> to counters). If you write `_total` in the name yourself, the result gets mangled and the
> metric will show up under an unexpected name.

---

## Viewing logs (Grafana + Loki)

1. Open Grafana at **http://localhost:3000** (user `admin`, password `admin`)
2. Add Loki as a *data source*: `http://loki:3100`
3. Go to **Explore** and query with LogQL, e.g.: `{job="docker"}`

---

## Screenshots

Save your images under a `docs/` folder and remove the `<!-- -->` comment markers below to make
them visible.

**Prometheus — querying `orders_created_total`:**

<!-- ![Prometheus query](docs/prometheus-query.png) -->

**Grafana — dashboard:**

<!-- ![Grafana dashboard](docs/grafana-dashboard.png) -->

---

## Key configuration

**`application.yaml`** — exposes the required Actuator endpoints:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
```

**`prometheus/prometheus.yml`** — tells Prometheus where and how often to scrape the app:

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: 'observability-demo'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['host.docker.internal:8081']
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Prometheus shows nothing | The screen starts empty | Type a metric and click **Execute**, on the **Table** tab |
| A custom metric shows up empty | Name differs from expected (`_total` suffix) | Check the real name at `http://localhost:8081/actuator/prometheus` |
| Target is `down` in Prometheus | The app isn't running or the port doesn't match | Check `http://localhost:8081/actuator/health` |
| The counter doesn't increase | You haven't called the endpoint, or the scrape hasn't run yet | Make the request and wait ~5 s (the `scrape_interval`) |

To verify Prometheus is scraping correctly, open **http://localhost:9090/targets** — the
`observability-demo` target should appear as **UP**.

---

## Stopping everything

```bash
# Stop the containers
docker compose down

# Stop and also remove volumes (deletes Prometheus/Loki historical data)
docker compose down -v
```

---

## Academic context

Individual project for the **Software Architecture (ARSW)** course —
Escuela Colombiana de Ingeniería Julio Garavito.
