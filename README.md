# Flashcastr Grafana Stack

Monitoring and observability stack for the Flashcastr/Space Invaders services. Deploys Prometheus and Grafana on Railway to monitor the producer (invaders.bot) and consumer (invaders.consumer) services.

## Services Monitored

- **invaders.bot (Producer)** - Syncs flash data from Space Invaders API, publishes to RabbitMQ and Farcaster
- **invaders.consumer (Consumer)** - Processes flashes, uploads images to IPFS, updates PostgreSQL

## Dashboards

### Service Status
Health and system metrics for both services:
- Service uptimes
- Circuit breaker status
- Memory usage (heap & RSS)
- API call success/error rates
- Error totals and consecutive failures
- Last activity timestamps

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

## Deployment

### Railway

1. Deploy this repo on Railway
2. Set environment variables for Prometheus:
   ```
   INVADERS_BOT_TARGET=producer.railway.internal:9090
   INVADERS_CONSUMER_TARGET=consumer.flashcastr.app
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
│   └── dockerfile        # Custom image with env var substitution
├── grafana/
│   ├── dashboards/
│   │   ├── service-status.json
│   │   └── flashes.json
│   ├── provisioning/
│   │   ├── dashboards/
│   │   └── datasources/
│   └── dockerfile
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

## Customization

### Adding Panels

1. Edit the dashboard JSON in `grafana/dashboards/`
2. Use `grafana_prometheus` as the datasource UID
3. Commit and redeploy

### Changing Scrape Targets

Update the environment variables in Railway:
- `INVADERS_BOT_TARGET` - Producer metrics URL
- `INVADERS_CONSUMER_TARGET` - Consumer metrics URL

---

Based on the [Grafana Stack Railway Template](https://railway.com/template/8TLSQD)
