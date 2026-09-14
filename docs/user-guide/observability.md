# Observability & Metrics

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

**Total route table size:**
raven_route_table_size                           # Total pre-policy routes in the table

**BMP session health:**
raven_bmp_session_state{router="192.168.1.1"}                     # 1=up, 0=down
raven_bmp_messages_total{router="192.168.1.1", msg_type="route_monitoring"}
raven_bmp_peer_state{router="192.168.1.1", peer="192.0.2.1"}      # 1=established, 0=down

**RTR cache health:**
raven_rtr_session_state{cache="localhost:3323"}      # 1=up, 0=down
raven_rtr_vrp_count{cache="localhost:3323"}          # Total VRPs loaded
raven_rtr_aspa_count{cache="localhost:3323"}         # Total ASPA records loaded
raven_rtr_last_sync_seconds{cache="localhost:3323"}  # Unix timestamp of last sync
raven_rtr_serial_number{cache="localhost:3323"}      # Current RTR serial
raven_rtr_cache_stale{cache="localhost:3323"}        # 1=stale, 0=fresh
raven_rtr_sync_duration_seconds{cache="localhost:3323"}   # Histogram of sync durations

**RTR anomaly detection:**

Emitted by the adaptive anomaly detector in `raven rtr monitor` when it is run
with `--prometheus`. `severity` is `high` or `medium`. See
[CLI Reference → raven rtr monitor](cli-reference.md#raven-rtr-monitor).

raven_rtr_anomaly_total{cache="localhost:3323", severity="high"}     # Count of anomalies, by severity
raven_rtr_anomaly_last_timestamp{cache="localhost:3323"}             # Unix timestamp of most recent anomaly

**External global-visibility correlation:**

Emitted when the Event Engine's `global-correlate` action runs, which requires
`external.ripestat.enabled: true`. `result` is the consensus verdict:
`match`, `divergent`, `local_only` or `inconclusive`. See
[Configuration → External Correlation](configuration.md#external-correlation-ripestat).

raven_global_check_total{source="ripestat", result="match"}          # Count of correlations, by source and verdict
raven_global_check_latency_seconds                                   # Histogram of correlation latency (live lookups only)
raven_global_check_cache_hits_total{source="ripestat"}               # Correlations served from the in-process cache
raven_global_check_rate_limited_total{source="ripestat"}             # Lookups suppressed by the local rate limiter

`raven_global_check_total` is a counter vector, so a given `result` series
does not appear until that verdict has occurred at least once.

`raven_global_check_latency_seconds` covers only correlations that actually
went to the provider. Cache hits and rate-limited lookups make no network
round-trip, so their near-zero timings are excluded — otherwise a warm cache
would pull p50/p95 toward zero and mask the provider latency degradation you
would want to alert on. Those two are counted on
`raven_global_check_cache_hits_total` and
`raven_global_check_rate_limited_total` instead, so the activity stays
visible.

A rate-limited lookup is a local policy decision, not a failure, so it does
not appear on `raven_global_check_total` at all. A cache hit does: it
produced a real verdict.

Cache hit ratio:

```promql
rate(raven_global_check_cache_hits_total[5m])
/
rate(raven_global_check_total[5m])
```

!!! note
    `raven check global` runs in its own short-lived CLI process, so its
    correlations are never scraped. These metrics reflect daemon-side
    correlations only.

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

      - alert: RAVENRTRAnomaly
        expr: increase(raven_rtr_anomaly_total{severity="high"}[10m]) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "RAVEN detected a high-severity RTR sync anomaly"

      - alert: RAVENGlobalDivergence
        expr: increase(raven_global_check_total{result="divergent"}[10m]) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "Global BGP view disagrees with RAVEN's local origin — likely a propagated hijack"

      - alert: RAVENGlobalCheckInconclusive
        expr: |
          increase(raven_global_check_total{result="inconclusive"}[30m])
          /
          increase(raven_global_check_total[30m]) > 0.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "More than half of global-visibility correlations are failing — check RIPEstat reachability"
```

**Global correlation verdict rate:**

```promql
sum by (result) (increase(raven_global_check_total[1h]))
```

A high `local_only` rate is worth investigating on its own: those prefixes
exist in your RIB but no route collector carries them, which points at leaks
or misconfiguration rather than hijacks.

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

**RTR Anomaly Detection**

- Total RTR Anomalies — `raven_rtr_anomaly_total`, broken down by severity
- Time Since Last Anomaly — driven by `raven_rtr_anomaly_last_timestamp`

**BMP Session Health**

- Messages per second by router
- Route count per peer
- Session up/down events

**Global BGP Visibility Correlation**

The last row on the dashboard, after RTR Anomaly Detection. It is fed by the
Event Engine's `global-correlate` action, so it stays empty until
`external.ripestat.enabled: true` and a rule using that action fires. See
[Configuration → External Correlation](configuration.md#external-correlation-ripestat).

- Global Consensus Results — `rate(raven_global_check_total[5m])` by
  `result`. Colour-matched to the security posture palette: match green,
  divergent red, local_only amber, inconclusive grey.
- Global Consensus — Current Totals — one stat per verdict, straight off the
  `raven_global_check_total` counters. Reads `0` rather than "No data" before
  the first correlation runs, so a verdict appearing is visible as a change.
- RIPEstat Query Performance — cache hits
  (`raven_global_check_cache_hits_total`), rate-limited calls
  (`raven_global_check_rate_limited_total`) and live queries
  (`raven_global_check_latency_seconds_count`) on one graph. This is the
  panel to watch when tuning `cache-ttl` and `rate-limit-per-min`: rising
  rate-limited calls mean the budget is too tight for your event volume.
- RIPEstat Query Latency p50/p95 — quantiles over
  `raven_global_check_latency_seconds`, in seconds. A p95 spike means
  RIPEstat is slow or unreachable, not that RAVEN is busy.

This row answers the question the posture panels cannot. When a prefix turns
up as origin-invalid, the verdict here tells you how far the announcement
actually spread. `local_only` ticking up on Current Totals means no route
collector carries the prefix at all — the announcement never left your
network, which points at a leak, a misconfiguration or a lab injection rather
than a hijack the internet has accepted. `divergent` is the opposite and the
more serious reading: the world sees a different origin than you do, so the
bogus announcement has propagated. Global Consensus Results shows the same
verdict as a rate over time, so you can see when it started.

!!! note
    The latency panels deliberately do not filter by `{source="ripestat"}`.
    `raven_global_check_latency_seconds` is registered as a plain histogram
    rather than a histogram vector, so it carries no `source` label — adding
    the matcher would select no series and leave both panels empty.

!!! tip
    `lab/demo-master.sh setup` imports the dashboard through the Grafana API
    with `overwrite: true`, so the demo lab picks this row up automatically —
    no manual import step. See [Lab → Overview](../lab/overview.md).

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
