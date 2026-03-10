# Flashcastr Grafana Stack

Monitoring and observability stack for the Flashcastr microservices pipeline.

## Architecture

```
flashcastr.services (5 services)
    ├── metrics ──> Prometheus (:9090) ──> Grafana (:3000)
    ├── traces  ──> Tempo (:3200)     ──> Grafana
    └── logs    ──> Loki (:3100)      ──> Grafana
```

Tempo also generates RED metrics and pushes them to Prometheus via remote write.

## Stack

- **Prometheus** — metrics scraping (5 service targets via env vars)
- **Loki 3.4** — log aggregation (TSDB schema, filesystem storage)
- **Tempo 2.9.0** — distributed tracing (OTLP gRPC :4317, HTTP :4318)
- **Grafana 11.5.2** — dashboards and visualization

## Dashboards (7 total)

All in `grafana/dashboards/`, auto-provisioned:

| File | Purpose |
|------|---------|
| `overview.json` | Home — service health, memory, trace-derived RED metrics |
| `api.json` | GraphQL requests, durations, cache hits/misses |
| `flash-engine.json` | Flash fetching pipeline |
| `database-engine.json` | Batch insert stats |
| `image-engine.json` | IPFS pinning, circuit breaker |
| `neynar-engine.json` | Farcaster casting metrics |
| `logs.json` | Loki log viewer with service/level filters |

## Datasource UIDs

| Datasource | UID | Default |
|------------|-----|---------|
| Loki | `grafana_lokiq` | Yes |
| Prometheus | `grafana_prometheus` | No |
| Tempo | `grafana_tempo` | No |

## Commands

```bash
docker-compose up -d    # Start full stack locally
# Prometheus: http://localhost:9090
# Loki:       http://localhost:3100
# Tempo:      http://localhost:3200
# Grafana:    http://localhost:3000 (admin/admin)
```

## Environment Variables

Prometheus scrape targets (set in docker-compose or Railway):
- `FLASHCASTR_API_TARGET`
- `FLASH_ENGINE_TARGET`
- `DATABASE_ENGINE_TARGET`
- `IMAGE_ENGINE_TARGET`
- `NEYNAR_ENGINE_TARGET`

## Key Files

| File | Purpose |
|------|---------|
| `prometheus/prom.yml` | Scrape config with env var placeholders |
| `prometheus/dockerfile` | sed-based env substitution at runtime |
| `tempo/tempo.yml` | OTLP receivers + metrics generator config |
| `loki/loki.yml` | TSDB schema, in-memory ring, filesystem storage |
| `grafana/grafana.ini` | Sets overview.json as home dashboard |
| `grafana/datasources/datasources.yml` | Prometheus, Loki, Tempo datasource definitions |
