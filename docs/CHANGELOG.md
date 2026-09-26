# Changelog

All notable changes to RAVEN are recorded here.

## Unreleased

### Added
- On-demand global BGP visibility correlation via the RIPEstat Data API.
  RAVEN's ROV/ASPA validation is local-vantage-point only and cannot
  distinguish a globally propagated hijack from a purely local leak; this
  compares the local BMP-observed origin for a prefix against the origin
  ASNs that RIS route collectors see for it. Verdicts are `match`,
  `divergent`, `local_only` or `inconclusive`.
- `raven check global --prefix <cidr> [--origin-asn N] [--format table|json]`
  — one-shot correlation showing the local BMP view beside the global view
  with a consensus verdict. Works without a running daemon when
  `--origin-asn` is supplied.
- New `external.ripestat` config section (`enabled`, `base-url`, `timeout`,
  `cache-ttl`, `rate-limit-per-min`), defaulting to disabled. Existing
  `raven.yaml` files are unaffected without an explicit opt-in.
- New Event Engine action type `global-correlate`, configurable per rule
  with an optional `cache_ttl`. It runs before the rule's other actions and
  annotates the event, so webhook payloads gain a `global_visibility` field
  and the `log` action gains `global_*` keys. Rules that do not use it are
  unaffected. Correlation runs in the Event Engine's own goroutines behind a
  per-prefix cache, a token-bucket rate limiter and a concurrency bound —
  never on the BMP ingest or validation path, and with no per-route external
  HTTP calls.
- New Prometheus metrics: `raven_global_check_total{source,result}`,
  `raven_global_check_latency_seconds`,
  `raven_global_check_rate_limited_total{source}` and
  `raven_global_check_cache_hits_total{source}`. The last two count the
  lookups that made no network round-trip — suppressed by the local rate
  limiter, or served from the in-process cache — and are deliberately kept
  off the latency histogram, which covers live lookups only. A policy
  decision and a warm cache must both stay distinguishable from a provider
  RAVEN could not reach.

  The result is a standalone annotation: it does not feed into
  `SecurityPosture` and the ROV × ASPA posture matrix is unchanged. The
  whole path is fail-open — an unreachable, slow or malformed RIPEstat
  degrades to `inconclusive` and never crashes RAVEN or blocks an action.
- Grafana: Global BGP Visibility Correlation dashboard row — consensus
  results timeseries, current totals stat panel, RIPEstat query performance,
  and latency p50/p95 panels (panels 17–20).
- RTR anomaly detection: adaptive median/MAD-based detector for RTR sync
  telemetry (interval, duration, VRP/ASPA churn) with per-cache rolling
  baselines, hard-trip and correlated-trip classification.
- Restart-safe baseline persistence via `--anomaly-snapshot` on
  `raven rtr monitor` — anomaly detector windows survive process restarts.
- `raven rtr seed-baseline` command to seed a detector baseline from
  historical NDJSON telemetry, avoiding the ~25-hour warm-up window on
  every fresh deployment.
- New Prometheus metrics: `raven_rtr_anomaly_total`,
  `raven_rtr_anomaly_last_timestamp`.
- `lab/04-rtr-anomaly.sh` Containerlab demo scenario for live RTR
  anomaly detection (bulk SLURM ROA injection, serial-based confirmation).

### Fixed
- A `raven.yaml` that exists but does not parse is now fatal. RAVEN prints
  the error (naming the offending file) to stderr and exits 1. Previously a
  YAML syntax error printed one line and then started the daemon anyway on
  silently-defaulted config — `rtr_caches:0`, no BMP peers, no event rules,
  exit 0 — so a single mis-indented key left an operator with a daemon that
  looked healthy and validated nothing. A missing config file stays
  non-fatal: RAVEN still runs on defaults when no `raven.yaml` exists.
- Global-visibility lookups suppressed by the local rate limiter are no
  longer reported as queries. The result now carries `queried: false` and
  no `latency_ns`, instead of `queried: true` with a sub-microsecond
  latency for a call that never left the process. They are counted on the
  new `raven_global_check_rate_limited_total{source}` rather than folded
  into `raven_global_check_total{result="inconclusive"}`, and contribute no
  observation to `raven_global_check_latency_seconds`, which now describes
  only real network attempts. In Prometheus and Grafana, "the rate limiter
  fired" and "RIPEstat was unreachable" are now separate signals.
- CLI errors are now printed to stderr. The root command sets
  `SilenceErrors`, so cobra did not print returned errors and `main`
  discarded them — every CLI failure exited 1 with no output at all,
  including config validation errors and commands' actionable hints.
- RTR anomaly detector no longer evaluates or contaminates its baseline
  with full (non-incremental) RTR syncs, which previously produced a
  false-positive high-severity anomaly on every `raven rtr monitor`
  startup.
- Upstream ASPA verification (`internal/validation/aspa`) now checks the
  first hop of the AS_PATH — whether the immediate peer AS lists the
  monitoring router's own AS as an authorised provider. This check
  previously stopped one hop short, so a real route-propagation
  misconfiguration over a peering session without a matching ASPA
  provider entry resolved as `ASPA:Unknown` instead of `ASPA:Invalid`.
  Single-AS-path routes, previously short-circuited to `Unknown` with no
  check at all, are now actually evaluated.
- Adds `LocalASN` to `Route`, learned per BMP session from the Peer Up
  message's Sent OPEN (RFC 6793 four-octet AS capability preferred when
  present) via new `parseOpenLocalASN()` in `internal/bmp/parser.go`.
  No config changes needed.

### Changed
- Any direct BGP session where the peer's ASPA record does not list the
  local AS as a provider now shows `path-suspect` for every prefix
  received over that session — not just customer-originated
  misannouncements, but connected-link and infrastructure prefixes too.
  This is correct per draft-ietf-sidrops-aspa-verification, not a
  regression; see
  [Security Postures](user-guide/security-postures.md#path-suspect).

## v0.3.3 (2026-07-02)

### Added
- RTR session telemetry: structured per-sync event capture (VRP/ASPA
  announced/withdrawn deltas, sync duration, interval between serial
  advances, sync type full vs incremental) wired into the RTR client
  at four points: session connect, EndOfData, PDUCacheReset, and
  PDUErrorReport. Events written as NDJSON and mirrored on a buffered
  channel for downstream consumers.
- `raven rtr monitor` command: standalone RTR cache observer for
  baseline data collection and diagnostics. Config-file-free (flags
  only: `--cache`, `--log-file`, `--transport`, `--prometheus`).
  Graceful shutdown with bounded 70s grace period. Optional Prometheus
  endpoint. Useful for characterising normal RTR behaviour before
  deploying full RAVEN.
- `tls-min-version` config option for RTR and BMP TLS transports.
  Accepts `1.2` or `1.3`; defaults to TLS 1.2 minimum. Addresses
  operator transport security requirements (Orange/AS3215).

### Fixed
- RTR client: 65s read deadline with benign-timeout handling prevents
  shutdown from blocking indefinitely on idle caches while avoiding
  spurious reconnects on healthy sessions.
- RTR client: proto version reset to configured starting version on
  each reconnect, preventing permanent version downgrade caused by
  transient errors during cache startup.
- Telemetry event channel drained in standalone `rtr monitor` mode
  to prevent buffer fill and drop warnings during sustained reconnect
  loops.

### Changed
- Grafana: BGP Peers panel filtered to IPv4-only peers, height
  increased to accommodate IPv6 peer additions.

## v0.3.2 (2026-06-16)

### Fixed
- Demo lab: all `docker exec` calls in `demo-master.sh` now use `sudo` (silent
  injection failures occurred when run without root)
- Demo lab: `setup()` now injects `LEAK-PREFIX` prefix-list into running FRR
  containers via `vtysh` post-deploy; Containerlab mounts `frr.conf` but FRR
  does not reload on redeploy, causing the leak scenario to fail on cold start
- Demo lab: Routinator 0.15.1 changed CLI syntax; `--config` flag must now be
  passed as a global flag before the subcommand
  (`routinator -c ~/.routinator.conf server`) not after it
- Grafana: BGP Peers panel filtered to IPv4-only peers and panel height
  increased (IPv6 peer additions doubled tile count, causing overflow)

## v0.3.1 (2026-06-08)

### Added
- IPv6 route monitoring via BMP (`MP_REACH_NLRI` / `MP_UNREACH_NLRI` parsing)
- IPv6 ROV validation
- IPv6 origin hijack scenario in demo lab (`./demo-master.sh hijack6`)
- TLS support for BMP listener and RTR client; skip TCP buffer tuning
  (`SetReadBuffer`/`SetWriteBuffer`) for TLS connections (WSL2 compatibility)

### Fixed
- Demo lab: `lacnic` scenario 4 route leak used inline `LEAK-INJECT` route-map
  with wrong prepend ASN (AS64496 → ROV:Invalid); replaced with permanent
  `ROUTE-LEAK` route-map (prepend AS1199 → ROV:Valid, ASPA:Invalid,
  posture:path-suspect as intended)
- Demo lab: `leak` and `hijack6` vtysh commands converted to heredoc syntax
  (`docker exec ... bash -c "vtysh << 'VTYSH' ... VTYSH"`) to fix silent
  failures caused by leading spaces in `-c` arguments
- Demo lab: leak scenario prefix corrected to `193.0.0.0/21` (AS3333 /
  RIPE NCC); SLURM ASPA assertion added for AS3333 with provider AS1103
  (excluding AS65000 to trigger path-suspect)

### Changed
- Demo lab internet router ASN changed from AS2121 to AS64496 to avoid
  spurious `path-suspect` at baseline caused by real-world RPKI/ASPA
  records for AS2121

## v0.3.0 (2026-05-26)

### Added
- Event Engine: configurable triggers on posture changes
  (ROV state change, new route with specific posture,
  RTR cache failure) with webhook HTTP POST and file
  log actions
- Flowspec lifecycle management: detect origin-invalid
  route → generate Flowspec rule → inject via GoBGP →
  monitor → expire after configurable TTL. Dry-run mode
  and approval webhook supported.
- `raven audit` — full security posture report for a
  router: per-peer posture breakdown, ROV/ASPA coverage,
  recommendations. Outputs table, JSON, or markdown.
- `raven check stealthy` — detect stealthy BGP hijacks
  by comparing BMP control-plane view against data-plane
  forwarding via probes
- Warm-start persistence: snapshot route table and RPKI
  caches to disk on shutdown, restore on startup
- OpenTelemetry OTLP metrics export alongside Prometheus
- RTR-over-TLS: configure `transport: tls` and optional
  CA cert under any RTR cache entry
- BMP listener TLS: optional TLS termination on the BMP
  listener with mutual TLS support
- Event Engine ASN triggers: `asn` and `protected_asn`
  trigger types (contributed by Orange/AS3215)

### Fixed
- RTR `rtr-version` config field now correctly wired
  through to the client (was previously ignored)
- TCP socket buffer tuning skipped for TLS connections
  (prevented TLS sessions from establishing on some
  platforms)

## v0.2.0 (2026-05-18)

### Added
- ASPA path verification per draft-ietf-sidrops-aspa-verification-24
- Combined security posture matrix: Secured / Origin-Only / Path-Suspect /
  Path-Only / Unverified / Origin-Invalid
- ASPA store populated via RTR v2 ASPA PDUs; re-validation on ASPA
  store updates via dirty-set propagation
- `raven aspa` — show ASPA records for an ASN
- `raven aspa recommend` — suggest ASPA objects based on observed paths
- `raven what-if` — simulate impact of reject-invalid or ASPA enforcement
- `raven watch` — stream live validation state changes
- Prometheus posture metrics reflect full ROV × ASPA matrix

## v0.1.0 (2026-04-15)

Initial public release — Phase 1 (Foundation) complete.

### BMP Ingest
- Embedded BMP receiver (RFC 7854) on configurable TCP port (default: 11019)
- Parses BGP UPDATE messages from BMP Route Monitoring PDUs
- Supports Adj-RIB-In Pre-Policy, Post-Policy, and Loc-RIB
- Per-session lifecycle management (Initiation, Peer Up/Down, Termination)

### RPKI / RTR Client
- RTR v1 (RFC 8210) and RTR v2 (draft-ietf-sidrops-8210bis) client
- VRP store for Route Origin Validation
- Multi-cache support with preference ordering and automatic failover
- Re-validation on RPKI cache updates via dirty-set propagation

### Validation Engine
- Route Origin Validation (ROV) per RFC 6811: Valid / Invalid / NotFound

### CLI
- `raven serve` — start daemon
- `raven status` — BMP peer and RTR cache health
- `raven peers` — list BMP peers
- `raven routes` — query route table with filters (prefix, origin-asn, peer, posture)
- `raven validate` — one-shot prefix validation

### Observability
- Prometheus metrics endpoint (default: 9595)
- Pre-built Grafana dashboards (Security Posture Overview, Per-Peer Deep Dive)

### Demo Lab
- Containerlab topology: internet AS64496 → upstream AS65000 → edge AS65001
- Scripted demo scenarios: origin hijack, route leak
