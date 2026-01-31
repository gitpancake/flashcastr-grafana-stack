# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with this repository.

## Project Overview

Flashcastr Grafana Stack is a monitoring and observability solution for the Flashcastr/Space Invaders services. It deploys Prometheus and Grafana on Railway to scrape metrics from the producer (invaders.bot) and consumer (invaders.consumer) services.

## Architecture

```
┌─────────────────────┐     ┌─────────────────────┐
│   invaders.bot      │     │  invaders.consumer  │
│   (Producer)        │     │     (Consumer)      │
│   Port 9090         │     │     Port 9091       │
└──────────┬──────────┘     └──────────┬──────────┘
           │                           │
           └───────────┬───────────────┘
                       ▼
              ┌─────────────────┐
              │   Prometheus    │
              │   (Scraper)     │
              └────────┬────────┘
                       ▼
              ┌─────────────────┐
              │    Grafana      │
              │  (Dashboards)   │
              └─────────────────┘
```

## Project Structure

```
flashcastr-grafana-stack/
├── prometheus/
│   ├── prom.yml              # Prometheus config with scrape targets
│   └── dockerfile            # Custom Prometheus image with env var substitution
├── grafana/
│   ├── dashboards/
│   │   ├── service-status.json   # Service health, uptime, memory, errors
│   │   └── flashes.json          # Flash processing metrics
│   ├── provisioning/
│   │   ├── dashboards/
│   │   │   └── dashboards.yml    # Dashboard provisioning config
│   │   └── datasources/
│   │       └── datasources.yml   # Prometheus datasource config
│   └── dockerfile            # Custom Grafana image with dashboards
└── docker-compose.yml        # Local development setup
```

## Dashboards

### Service Status Dashboard
- Service uptimes (Producer & Consumer)
- Circuit breaker status
- Consecutive failures
- Last activity timestamps
- Memory usage (heap & RSS)
- Uptime history graphs
- API call rates with success/error breakdown
- Error totals

### Flashes Processing Dashboard
- New flashes count (Producer)
- Flashes processed count (Consumer)
- Total flash count from API
- IPFS uploads total
- Processing rates per minute
- Sync skip reasons (no_changes, backoff, off_peak_hours)
- Duration histograms (p50/p90/p99)
- Queue depth
- RabbitMQ messages published/failed
- Farcaster casts published/failed

## Environment Variables

The Prometheus dockerfile uses sed-based substitution for these variables:

- `INVADERS_BOT_TARGET` - Producer metrics endpoint (e.g., `producer.railway.internal:9090` or `invadersbot-production.up.railway.app`)
- `INVADERS_CONSUMER_TARGET` - Consumer metrics endpoint (e.g., `consumer.flashcastr.app`)

## Deployment on Railway

1. Deploy from this repo on Railway
2. Set environment variables for the Prometheus service:
   - `INVADERS_BOT_TARGET` - Internal or public URL for producer
   - `INVADERS_CONSUMER_TARGET` - Internal or public URL for consumer
3. The Grafana dashboards are automatically provisioned

## Local Development

```bash
# Start the stack locally
docker-compose up -d

# Access services
# Prometheus: http://localhost:9090
# Grafana: http://localhost:3000 (admin/admin)
```

## Metrics Endpoints

### Producer (invaders.bot) - Port 9090
- `invaders_bot_flashes_new_total` - New flashes stored
- `invaders_bot_api_calls_total{result}` - API calls (success/error)
- `invaders_bot_sync_skipped_total{reason}` - Sync skips by reason
- `invaders_bot_consecutive_unchanged_syncs` - Consecutive unchanged syncs
- `invaders_bot_last_flash_count` - Total flash count from API
- `invaders_bot_sync_duration_seconds` - Sync duration histogram
- `invaders_bot_messages_published_total` - RabbitMQ messages published
- `invaders_bot_casts_published_total` - Farcaster casts published
- `invaders_bot_uptime_seconds` - Service uptime
- `invaders_bot_memory_bytes{type}` - Memory usage

### Consumer (invaders.consumer) - Port 9091
- `invaders_consumer_flashes_processed_total` - Flashes processed
- `invaders_consumer_flashes_failed_total` - Failed processing
- `invaders_consumer_ipfs_uploads_total` - IPFS uploads
- `invaders_consumer_ipfs_failures_total` - IPFS failures
- `invaders_consumer_processing_duration_seconds` - Processing duration histogram
- `invaders_consumer_queue_depth` - RabbitMQ queue depth
- `invaders_consumer_circuit_breaker_state` - Circuit breaker (0=closed, 1=open)
- `invaders_consumer_consecutive_failures` - Consecutive failures
- `invaders_consumer_uptime_seconds` - Service uptime
- `invaders_consumer_memory_bytes{type}` - Memory usage

## Adding New Panels

1. Edit the appropriate dashboard JSON file in `grafana/dashboards/`
2. Use `grafana_prometheus` as the datasource UID
3. Follow the existing panel structure for consistency
4. Commit and redeploy
