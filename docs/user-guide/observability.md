# Prometheus & Grafana

RAVEN exposes a Prometheus metrics endpoint and ships with pre-built Grafana
dashboards that give you a real-time view of your routing security posture.

## Prometheus

### Enabling the Metrics Endpoint

Add to your `raven.yaml`:

```yaml
outputs:
  prometheus:
    listen: ":9595"
    path: "/metrics"
```

Verify it is working:

```bash
curl http://localhost:9595/metrics | grep raven
```

### Prometheus Scrape Configuration

Add RAVEN to your `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: raven
    static_configs:
      - targets: ["localhost:9595"]
    scrape_interval: 15s
```

!!! note
    If RAVEN is running in a container or WSL and Prometheus is running in
    Docker, use the Docker bridge gateway IP instead of localhost.
    Typically `172.17.0.1` on Linux:

```yaml
    static_configs:
      - targets: ["172.17.0.1:9595"]
```

### Available Metrics

**Route counts by security posture:**
raven_routes_total{posture="secured", afi="ipv4"}
raven_routes_total{posture="origin-only", afi="ipv4"}
raven_routes_total{posture="path-suspect", afi="ipv4"}
raven_routes_total{posture="path-only", afi="ipv4"}
raven_routes_total{posture="unverified", afi="ipv4"}
raven_routes_total{posture="origin-invalid", afi="ipv4"}

Same metrics available with `afi="ipv6"`.

**Per-peer route counts:**
raven_peer_routes{peer="192.168.1.1", peer_asn="65001", posture="secured"}
raven_peer_routes{peer="192.168.1.1", peer_asn="65001", posture="origin-invalid"}

**BMP session health:**
raven_bmp_session_state{router="192.168.1.1"}    # 1=up, 0=down
raven_bmp_messages_total{router="192.168.1.1", type="route_monitoring"}
raven_bmp_peer_routes{router="192.168.1.1"}

**RTR cache health:**
raven_rtr_session_state{cache="localhost:3323"}      # 1=up, 0=down
raven_rtr_vrp_count{cache="localhost:3323"}          # Total VRPs loaded
raven_rtr_aspa_count{cache="localhost:3323"}         # Total ASPA records loaded
raven_rtr_last_sync_seconds{cache="localhost:3323"}  # Unix timestamp of last sync
raven_rtr_serial_number{cache="localhost:3323"}      # Current RTR serial
raven_rtr_cache_stale{cache="localhost:3323"}        # 1=stale, 0=fresh
raven_rtr_sync_duration_seconds{cache="localhost:3323"}

### Useful PromQL Queries

**Percentage of routes that are origin-invalid:**

```promql
raven_routes_total{posture="origin-invalid", afi="ipv4"}
/
sum(raven_routes_total{afi="ipv4"}) * 100
```

**Alert: origin-invalid routes above threshold:**

```yaml
groups:
  - name: raven
    rules:
      - alert: RAVENOriginInvalidHigh
        expr: raven_routes_total{posture="origin-invalid"} > 100
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High number of origin-invalid routes detected"

      - alert: RAVENPathSuspect
        expr: raven_routes_total{posture="path-suspect"} > 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Path-suspect routes detected — possible route leak"

      - alert: RAVENRTRCacheStale
        expr: raven_rtr_cache_stale == 1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "RAVEN RTR cache is stale — RPKI validation may be outdated"

      - alert: RAVENBMPSessionDown
        expr: raven_bmp_session_state == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "RAVEN BMP session is down"
```

## Grafana

### Importing the Pre-built Dashboard

RAVEN ships with a Grafana dashboard in the repository at
`lab/grafana-dashboard.json`.

**Via Grafana UI:**

1. Go to **Dashboards → Import**
2. Click **Upload JSON file**
3. Select `lab/grafana-dashboard.json` from the RAVEN repository
4. Set the Prometheus datasource
5. Click **Import**

**Via Grafana API:**

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -d @lab/grafana-dashboard.json \
  http://admin:admin@localhost:3000/api/dashboards/import
```

### Dashboard Panels

The pre-built dashboard includes:

**Security Posture Overview**

- Route count by posture (time series) — watch for spikes in origin-invalid
- Posture distribution (pie chart) — your network's overall security health
- Origin-invalid routes table — prefix, peer, origin ASN, matched VRP
- Path-suspect routes table — prefix, peer, AS_PATH, failing hop

**RTR Cache Health**

- VRP count over time — should be stable, drops indicate validator issues
- ASPA count over time
- Cache sync latency
- Session state — alert panel turns red if cache goes down

**BMP Session Health**

- Messages per second by router
- Route count per peer
- Session up/down events

### Running Grafana with Docker

For the demo lab or a quick local setup:

```bash
docker run -d \
  --name grafana \
  -p 3000:3000 \
  -e GF_SECURITY_ADMIN_PASSWORD=admin \
  grafana/grafana:latest
```

Then import the dashboard as described above and point the Prometheus
datasource at your RAVEN metrics endpoint.
