# Flashcastr Grafana Stack

Monitoring and observability stack for the Flashcastr microservices pipeline. Deploys Prometheus, Loki, Tempo, and Grafana for metrics, logs, distributed tracing, and dashboards.

## Services Monitored

| Service | Description | Metrics Port |
|---------|-------------|--------------|
| **flash-engine** | Cron-fetches flashes from Space Invaders API | configurable |
| **image-engine** | Downloads images, pins to IPFS via Pinata | configurable |
| **database-engine** | Batch inserts flashes into Postgres | configurable |
| **neynar-engine** | Casts to Farcaster via Neynar SDK | configurable |
| **api** | GraphQL API for Flashcastr app | configurable |

## Dashboards

7 auto-provisioned dashboards in `grafana/dashboards/`:

| Dashboard | File | Description |
|-----------|------|-------------|
| **Overview** | `overview.json` | Home dashboard — service health, memory, API metrics, distributed traces |
| **API** | `api.json` | GraphQL request metrics, durations, cache performance |
| **Flash Engine** | `flash-engine.json` | Flash fetching and publishing metrics |
| **Image Engine** | `image-engine.json` | Image downloads, IPFS pinning, circuit breaker status |
| **Database Engine** | `database-engine.json` | Batch insert metrics, pool stats |
| **Neynar Engine** | `neynar-engine.json` | Farcaster casting metrics, retry stats |
| **Logs** | `logs.json` | Loki log aggregation with service and level filters |

### Distributed Tracing (Tempo)

Tempo collects distributed traces via OpenTelemetry (OTLP) and generates RED metrics:

1. Services send traces via OpenTelemetry SDK to Tempo
2. Tempo metrics generator creates RED metrics from traces
3. Prometheus receives metrics via remote write
4. Grafana visualizes trace-derived metrics in the Overview dashboard

| Metric | Description |
|--------|-------------|
| `traces_spanmetrics_calls_total` | Request count by service |
| `traces_spanmetrics_latency_bucket` | Request duration histogram |
| `traces_service_graph_request_total` | Service-to-service call counts |

### Log Aggregation (Loki)

Services ship structured logs to Loki via the `@flashcastr/logger` library. The Logs dashboard provides filtering by service and log level.

## Deployment

### Railway

1. Deploy this repo on Railway
2. Set environment variables for Prometheus scrape targets:
   ```
   FLASHCASTR_API_TARGET=api-host:port
   FLASH_ENGINE_TARGET=flash-engine-host:port
   DATABASE_ENGINE_TARGET=database-engine-host:port
   IMAGE_ENGINE_TARGET=image-engine-host:port
   NEYNAR_ENGINE_TARGET=neynar-engine-host:port
   ```
3. Dashboards are automatically provisioned in Grafana

### Local Development

```bash
docker-compose up -d

# Prometheus: http://localhost:9090
# Loki:       http://localhost:3100
# Tempo:      http://localhost:3200
# Grafana:    http://localhost:3000 (admin/admin)
```

## Project Structure

```
├── prometheus/
│   ├── prom.yml          # Scrape config (5 service targets via env vars)
│   └── dockerfile        # Custom image with env var substitution & remote write
├── loki/
│   ├── loki.yml          # TSDB schema, filesystem storage
│   └── dockerfile        # grafana/loki:3.4
├── tempo/
│   ├── tempo.yml         # OTLP receivers, metrics generator → Prometheus remote write
│   └── dockerfile        # grafana/tempo:2.9.0
├── grafana/
│   ├── dashboards/       # 7 JSON dashboard files (auto-provisioned)
│   ├── datasources/      # Prometheus, Loki, Tempo datasource configs
│   ├── provisioning/     # Dashboard provisioning config
│   ├── grafana.ini       # Sets overview.json as home dashboard
│   └── dockerfile        # grafana/grafana-oss:11.5.2
├── examples/
│   └── api/              # Example Node.js service with metrics, tracing, and logging
├── docker-compose.yml    # Full local stack (Prometheus, Loki, Tempo, Grafana)
└── README.md
```

## Environment Variables

### Prometheus Scrape Targets

| Variable | Description |
|----------|-------------|
| `FLASHCASTR_API_TARGET` | API service metrics endpoint |
| `FLASH_ENGINE_TARGET` | Flash engine metrics endpoint |
| `DATABASE_ENGINE_TARGET` | Database engine metrics endpoint |
| `IMAGE_ENGINE_TARGET` | Image engine metrics endpoint |
| `NEYNAR_ENGINE_TARGET` | Neynar engine metrics endpoint |

### Service Configuration

| Variable | Description |
|----------|-------------|
| `TEMPO_HTTP_ENDPOINT` | Set on each service: `http://tempo-host:4318/v1/traces` |
| `LOKI_URL` | Set on each service for log shipping: `http://loki-host:3100` |

## Configuring Services

### Metrics

Each service exposes a `/metrics` endpoint via `prom-client`. Set `METRICS_PORT` on each service.

### Tracing

Set the `TEMPO_HTTP_ENDPOINT` environment variable on each service to send traces to Tempo via OTLP HTTP.

### Logging

Set the `LOKI_URL` environment variable on each service. The `@flashcastr/logger` library batches and ships logs to Loki automatically.

## Customization

### Adding Panels

1. Edit the dashboard JSON in `grafana/dashboards/`
2. Use `grafana_prometheus` as the datasource UID for metrics
3. Use `grafana_lokiq` as the datasource UID for logs
4. Commit and redeploy

---

Based on the [Grafana Stack Railway Template](https://railway.com/template/8TLSQD)
