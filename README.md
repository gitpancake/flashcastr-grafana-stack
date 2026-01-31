# Flashcastr Grafana Stack

Monitoring and observability stack for the Flashcastr/Space Invaders services. Deploys Prometheus, Tempo, and Grafana on Railway for metrics, distributed tracing, and dashboards.

## Services Monitored

| Service | Description | Metrics Port |
|---------|-------------|--------------|
| **invaders.bot** (Producer) | Syncs flash data from Space Invaders API, publishes to RabbitMQ and Farcaster | 9090 |
| **invaders.consumer** (Consumer) | Processes flashes, uploads images to IPFS, updates PostgreSQL | 9091 |
| **flashcastr.api** (API) | GraphQL API for Flashcastr app, user management, leaderboards | 9092 |

## Dashboards

### Service Status
Health and system metrics for all services:
- Service uptimes (Producer, Consumer, API)
- Circuit breaker status
- Memory usage (heap & RSS)
- API call success/error rates
- GraphQL request metrics and durations
- Cache performance (hits/misses)
- Neynar API call rates
- Error totals

### Flashes Processing
Flash processing pipeline metrics:
- New flashes discovered (Producer)
- Flashes processed (Consumer)
- IPFS uploads
- Processing rates per minute
- Sync skip reasons
- Duration histograms (p50/p90/p99)
- Queue depth
- Farcaster cast metrics

### Distributed Tracing (Tempo)
The Service Status dashboard includes trace-derived RED metrics:
- Request rate by service (from traces)
- Error rate by service (from traces)
- Request duration percentiles (p50/p90/p99)
- Service-to-service call graphs

Tempo receives traces via OpenTelemetry (OTLP) from the Producer and API services.

## Deployment

### Railway

1. Deploy this repo on Railway
2. Set environment variables for Prometheus:
   ```
   INVADERS_BOT_TARGET=producer.railway.internal:9090
   INVADERS_CONSUMER_TARGET=consumer.flashcastr.app
   FLASHCASTR_API_TARGET=api.flashcastr.app
   ```
3. Dashboards are automatically provisioned in Grafana

### Local Development

```bash
docker-compose up -d

# Prometheus: http://localhost:9090
# Grafana: http://localhost:3000 (admin/admin)
```

## Project Structure

```
├── prometheus/
│   ├── prom.yml          # Scrape configuration
│   └── dockerfile        # Custom image with env var substitution & remote write
├── tempo/
│   ├── tempo.yml         # Tempo config with metrics generator
│   └── dockerfile        # Tempo image
├── grafana/
│   ├── dashboards/
│   │   ├── service-status.json   # Health, memory, API metrics, tracing
│   │   └── flashes.json          # Flash processing pipeline
│   ├── provisioning/
│   │   ├── dashboards/
│   │   └── datasources/
│   ├── grafana.ini       # Default home dashboard config
│   └── dockerfile
├── loki/                 # Log aggregation (optional)
└── docker-compose.yml
```

## Metrics Reference

### Producer Metrics (Port 9090)

| Metric | Type | Description |
|--------|------|-------------|
| `invaders_bot_flashes_new_total` | Counter | New flashes stored in database |
| `invaders_bot_api_calls_total{result}` | Counter | API calls (success/error) |
| `invaders_bot_sync_skipped_total{reason}` | Counter | Skipped syncs (no_changes/backoff/off_peak_hours) |
| `invaders_bot_consecutive_unchanged_syncs` | Gauge | Consecutive syncs with no changes |
| `invaders_bot_last_flash_count` | Gauge | Total flash count from API |
| `invaders_bot_sync_duration_seconds` | Histogram | Sync operation duration |
| `invaders_bot_messages_published_total` | Counter | RabbitMQ messages published |
| `invaders_bot_casts_published_total` | Counter | Farcaster casts published |
| `invaders_bot_uptime_seconds` | Gauge | Service uptime |
| `invaders_bot_memory_bytes{type}` | Gauge | Memory usage |

### Consumer Metrics (Port 9091)

| Metric | Type | Description |
|--------|------|-------------|
| `invaders_consumer_flashes_processed_total` | Counter | Flashes successfully processed |
| `invaders_consumer_flashes_failed_total` | Counter | Failed flash processing |
| `invaders_consumer_ipfs_uploads_total` | Counter | Successful IPFS uploads |
| `invaders_consumer_ipfs_failures_total` | Counter | Failed IPFS uploads |
| `invaders_consumer_processing_duration_seconds` | Histogram | Processing duration |
| `invaders_consumer_queue_depth` | Gauge | RabbitMQ queue depth |
| `invaders_consumer_circuit_breaker_state` | Gauge | Circuit breaker (0=closed, 1=open) |
| `invaders_consumer_consecutive_failures` | Gauge | Consecutive failures |
| `invaders_consumer_uptime_seconds` | Gauge | Service uptime |
| `invaders_consumer_memory_bytes{type}` | Gauge | Memory usage |

### API Metrics (Port 9092)

| Metric | Type | Description |
|--------|------|-------------|
| `flashcastr_api_graphql_requests_total` | Counter | GraphQL requests by operation |
| `flashcastr_api_graphql_errors_total` | Counter | GraphQL errors by operation |
| `flashcastr_api_graphql_duration_seconds` | Histogram | Request duration |
| `flashcastr_api_signups_initiated_total` | Counter | Signup initiations |
| `flashcastr_api_signups_completed_total` | Counter | Completed signups |
| `flashcastr_api_users_deleted_total` | Counter | User deletions |
| `flashcastr_api_cache_hits_total{cache_name}` | Counter | Cache hits |
| `flashcastr_api_cache_misses_total{cache_name}` | Counter | Cache misses |
| `flashcastr_api_neynar_requests_total` | Counter | Neynar API calls |
| `flashcastr_api_active_users_total` | Gauge | Active users |
| `flashcastr_api_total_flashes` | Gauge | Total flashes |
| `flashcastr_api_uptime_seconds` | Gauge | Service uptime |
| `flashcastr_api_memory_bytes{type}` | Gauge | Memory usage |

## Distributed Tracing with Tempo

Tempo collects distributed traces from services via OpenTelemetry Protocol (OTLP).

### How It Works

1. **Services send traces**: Producer and API services use OpenTelemetry SDK to send traces to Tempo
2. **Tempo generates metrics**: The metrics generator creates RED metrics from traces
3. **Prometheus scrapes metrics**: Tempo pushes metrics to Prometheus via remote write
4. **Grafana visualizes**: Dashboard panels show trace-derived metrics

### Trace Metrics Generated

| Metric | Description |
|--------|-------------|
| `traces_spanmetrics_calls_total` | Request count by service |
| `traces_spanmetrics_latency_bucket` | Request duration histogram |
| `traces_service_graph_request_total` | Service-to-service call counts |

### Configuring Services

Set the `TEMPO_HTTP_ENDPOINT` environment variable on each service:

```
TEMPO_HTTP_ENDPOINT=http://tempo.railway.internal:4318/v1/traces
```

Currently configured services:
- **invaders.bot** (Producer) - sends traces
- **flashcastr.api** (API) - sends traces
- **invaders.consumer** (Consumer) - does not send traces (runs on DigitalOcean)

## Customization

### Adding Panels

1. Edit the dashboard JSON in `grafana/dashboards/`
2. Use `grafana_prometheus` as the datasource UID
3. Commit and redeploy

### Changing Scrape Targets

Update the environment variables in Railway:
- `INVADERS_BOT_TARGET` - Producer metrics URL
- `INVADERS_CONSUMER_TARGET` - Consumer metrics URL
- `FLASHCASTR_API_TARGET` - API metrics URL

---

Based on the [Grafana Stack Railway Template](https://railway.com/template/8TLSQD)
