# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with this repository.

## Project Overview

Flashcastr Grafana Stack is a monitoring and observability solution for the Flashcastr/Space Invaders services. It deploys Prometheus and Grafana on Railway to scrape metrics from all three services.

## Architecture

```
┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐
│   invaders.bot      │  │  invaders.consumer  │  │   flashcastr.api    │
│   (Producer)        │  │     (Consumer)      │  │      (API)          │
│   Metrics: 9090     │  │   Metrics: 9091     │  │   Metrics: 9092     │
│   Traces: OTLP      │  │   (No tracing)      │  │   Traces: OTLP      │
└──────────┬──────────┘  └──────────┬──────────┘  └──────────┬──────────┘
           │                        │                        │
           │ metrics                │ metrics                │ metrics
           └────────────────────────┼────────────────────────┘
                                    ▼
                           ┌─────────────────┐
                           │   Prometheus    │◄──── remote write
                           │   (Metrics)     │         │
                           └────────┬────────┘         │
                                    │                  │
           ┌────────────────────────┼──────────────────┼─────────┐
           │                        │                  │         │
           │ traces (OTLP)          │                  │         │
           │                        ▼                  │         │
           │               ┌─────────────────┐         │         │
           └──────────────►│     Tempo       │─────────┘         │
                           │  (Traces +      │                   │
                           │   Metrics Gen)  │                   │
                           └────────┬────────┘                   │
                                    │                            │
                                    ▼                            │
                           ┌─────────────────┐◄──────────────────┘
                           │    Grafana      │
                           │  (Dashboards)   │
                           └─────────────────┘
```

## Project Structure

```
flashcastr-grafana-stack/
├── prometheus/
│   ├── prom.yml              # Prometheus config with scrape targets
│   └── dockerfile            # Custom image with env var substitution & remote write receiver
├── tempo/
│   ├── tempo.yml             # Tempo config with metrics generator for RED metrics
│   └── dockerfile            # Tempo image
├── grafana/
│   ├── dashboards/
│   │   ├── service-status.json   # Health, memory, API metrics, distributed tracing
│   │   └── flashes.json          # Flash processing pipeline metrics
│   ├── datasources/
│   │   └── datasources.yml       # Prometheus, Tempo, Loki datasources
│   ├── provisioning/
│   │   └── dashboards/
│   │       └── dashboards.yml    # Dashboard provisioning config
│   ├── grafana.ini               # Default home dashboard config
│   └── dockerfile                # Custom Grafana image with dashboards
├── loki/                         # Log aggregation (optional)
│   ├── loki.yml
│   └── dockerfile
└── docker-compose.yml            # Local development setup
```

## Dashboards

### Service Status Dashboard
- Service uptimes (Producer, Consumer, API)
- Circuit breaker status
- Last activity timestamps
- Memory usage (heap & RSS) for all services
- Uptime history graphs
- Producer & Consumer error rates
- Flashcastr API metrics (requests, duration, errors)
- Active users and total flashes
- Cache performance (hits/misses)
- Neynar API call rates
- Error totals
- **Distributed Tracing (Tempo)**:
  - Request rate by service (from traces)
  - Error rate by service (from traces)
  - Request duration percentiles (p50/p90/p99)
  - Service-to-service call graphs

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

- `INVADERS_BOT_TARGET` - Producer metrics endpoint (e.g., `producer.railway.internal:9090`)
- `INVADERS_CONSUMER_TARGET` - Consumer metrics endpoint (e.g., `consumer.flashcastr.app`)
- `FLASHCASTR_API_TARGET` - API metrics endpoint (e.g., `api.flashcastr.app`)

### Tempo Environment Variables (on services)

Services that send traces need:

- `TEMPO_HTTP_ENDPOINT` - Tempo OTLP endpoint (e.g., `http://tempo.railway.internal:4318/v1/traces`)

Currently tracing-enabled:
- **invaders.bot** (Producer) - Yes
- **flashcastr.api** (API) - Yes
- **invaders.consumer** (Consumer) - No (runs on DigitalOcean, can't reach Railway internal)

## Deployment on Railway

1. Deploy from this repo on Railway
2. Set environment variables for the Prometheus service:
   - `INVADERS_BOT_TARGET`
   - `INVADERS_CONSUMER_TARGET`
   - `FLASHCASTR_API_TARGET`
3. The Grafana dashboards are automatically provisioned

## Local Development

```bash
docker-compose up -d

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

### API (flashcastr.api) - Port 9092
- `flashcastr_api_graphql_requests_total{operation_type, operation_name}` - GraphQL requests
- `flashcastr_api_graphql_errors_total{operation_type, operation_name}` - GraphQL errors
- `flashcastr_api_graphql_duration_seconds` - Request duration histogram
- `flashcastr_api_signups_initiated_total` - Signups started
- `flashcastr_api_signups_completed_total` - Signups completed
- `flashcastr_api_users_deleted_total` - Users deleted
- `flashcastr_api_cache_hits_total{cache_name}` - Cache hits
- `flashcastr_api_cache_misses_total{cache_name}` - Cache misses
- `flashcastr_api_neynar_requests_total{endpoint, status}` - Neynar API calls
- `flashcastr_api_active_users_total` - Active users (gauge)
- `flashcastr_api_total_flashes` - Total flashes (gauge)
- `flashcastr_api_uptime_seconds` - Service uptime
- `flashcastr_api_memory_bytes{type}` - Memory usage

### Tempo-Generated Metrics (from traces)
- `traces_spanmetrics_calls_total{service, status_code}` - Request count by service
- `traces_spanmetrics_latency_bucket{service, le}` - Request duration histogram
- `traces_service_graph_request_total{client, server}` - Service-to-service calls

## Adding New Panels

1. Edit the appropriate dashboard JSON file in `grafana/dashboards/`
2. Use `grafana_prometheus` as the datasource UID
3. Follow the existing panel structure for consistency
4. Commit and redeploy
