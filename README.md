# Background

Software industry benchmarks suggest that observability typically accounts for ~10–17% of total infrastructure spend, 300 dollars per month per engineer. This project is to show that with our coding agent, you can save your
observability cost to zero.

# Code Observability & SLA Dashboard

This repository contains a lightweight, built-in observability layer and SLA dashboard for the PickleGlass web backend and web UI. It instruments the Express API, aggregates request metrics in-memory, exposes metrics endpoints (JSON + Prometheus), and renders a Next.js dashboard for fast feedback while developing.

## What it measures

- Availability: ratio of successful responses (status < 400)
- Availability under SLA: successful responses with latency ≤ SLA threshold (default 500 ms)
- Error rate: (4xx + 5xx) / total
- Throughput: requests per second (windowed)
- Latency: p50 / p90 / p95 / p99 (windowed)

All metrics are computed over a configurable rolling window and kept in-memory (for prototyping and local debugging).

## Architecture

- Express middleware: collects per-request metrics (status, duration, path, method)
  - File: `glass/pickleglass_web/backend_node/middleware/metrics.js`
- Metrics API routes (JSON + Prometheus):
  - File: `glass/pickleglass_web/backend_node/routes/metrics.js`
  - Mounted in: `glass/pickleglass_web/backend_node/index.js`
- Next.js dashboard page:
  - Path: `glass/pickleglass_web/app/observability/page.tsx`
  - Linked from the sidebar under Settings → Observability

## Requirements

- Node.js 18+
- npm 9+

c### Quick start (stdio)

1) Install and build
```bash
cd /Users/alex/project/hackthon/code-observability/mcp-grafana
npm install
npm run build
```

2) Run (set envs as needed)
```bash
GRAFANA_URL=http://localhost:3000 \
PROM_URL=http://localhost:9090 \
GLASS_API=http://localhost:9001 \
GRAFANA_TOKEN=... \
node build/index.js
```

### Test with MCP Inspector
```bash
cd /Users/alex/project/hackthon/code-observability/mcp-grafana
npx @modelcontextprotocol/inspector build/index.js
```

### Use in Cursor (MCP stdio)
Settings → MCP servers:
```json
<code_block_to_apply_changes_from>
```

### Use in Claude Desktop (macOS)
Add to `~/Library/Application Support/Claude/claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "mcp-grafana": {
      "type": "stdio",
      "command": "node",
      "args": ["/Users/alex/project/hackthon/code-observability/mcp-grafana/build/index.js"],
      "env": {
        "GRAFANA_URL": "http://localhost:3000",
        "PROM_URL": "http://localhost:9090",
        "GLASS_API": "http://localhost:9001",
        "GRAFANA_TOKEN": ""
      }
    }
  }
}
```

Notes
- To power `glass.sla_summary`, ensure the Glass API runs on 9001:
  - cd `/Users/alex/project/hackthon/code-observability/glass/pickleglass_web` then `npm run dev:api`
- If Grafana requires auth, set `GRAFANA_TOKEN` (Bearer).

## API endpoints

Base: `http://localhost:9001`

- GET `/api/metrics/summary`
  - Query params:
    - `windowMs` (number, optional, default 86400000): lookback window in ms
    - `latencySlaMs` (number, optional, default 500)
  - Returns:
    - `counts`: `{ total, successes, errors4xx, errors5xx }`
    - `rates`: `{ errorRate, availability, availabilityUnderSla, rps }`
    - `latency`: `{ p50, p90, p95, p99 }`

- GET `/api/metrics/timeseries`
  - Query params:
    - `windowMs` (number, optional, default 3600000)
    - `bucketSeconds` (number, optional, default 60)
    - `latencySlaMs` (number, optional, default 500)
  - Returns an array of buckets with the same shape as `summary` fields, per bucket.

- GET `/api/metrics/prometheus`
  - Query params:
    - `windowMs` (number, optional, default 300000)
  - Returns Prometheus-compatible text exposing request totals and latency quantiles over the window.

Example cURL:

```
curl "http://localhost:9001/api/metrics/summary?windowMs=3600000&latencySlaMs=500"
curl "http://localhost:9001/api/metrics/timeseries?windowMs=3600000&bucketSeconds=60"
curl "http://localhost:9001/api/metrics/prometheus?windowMs=300000"
```

## Configuration

- In-memory retention cap: `MAX_RECORDS = 50000`
- Default SLA latency threshold: `DEFAULT_LATENCY_SLA_MS = 500`
- Both are defined in `glass/pickleglass_web/backend_node/middleware/metrics.js`

## Production considerations

- Persistence: The current store is in-memory and resets on restart. For production, export to Prometheus (scrape `/api/metrics/prometheus`) and/or ship spans/metrics via OpenTelemetry.
- Protection: Lock down `/api/metrics/*` behind auth or network policies. By default, these routes are public to simplify local dev.
- Overhead: The middleware captures timestamps and computes durations on response finish. Overhead is minimal, but validate for your traffic profile.
- Cardinality: Paths are recorded verbatim in-memory but not exported per-path. Extend exporter if you need per-route metrics; be mindful of label cardinality.

## Where things live

- Express app setup: `glass/pickleglass_web/backend_node/index.js`
- Metrics middleware: `glass/pickleglass_web/backend_node/middleware/metrics.js`
- Metrics routes: `glass/pickleglass_web/backend_node/routes/metrics.js`
- Dev API server: `glass/pickleglass_web/backend_node/devServer.js`
- Frontend page: `glass/pickleglass_web/app/observability/page.tsx`
- Frontend API helper: `glass/pickleglass_web/utils/api.ts`

## Troubleshooting

- API not reachable from the web app:
  - Ensure API is running on port 9001: `npm run dev:api`
  - Default CORS allows `http://localhost:3000`. Customize via `pickleglass_WEB_URL`.
- Prometheus scrape shows 0 values:
  - Generate some traffic against `/api/*` endpoints; the middleware records on response finish.
- Dev server fails with IPC errors:
  - That is expected for routes relying on the Electron bridge. Metrics, `/api/sync/status`, and other non-IPC routes work in dev.

## License

See `glass/LICENSE`. 
