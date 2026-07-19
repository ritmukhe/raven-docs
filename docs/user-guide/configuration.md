# Configuration

RAVEN is configured via a YAML file. Every section is optional — RAVEN uses
sensible defaults for anything not specified.

## Minimal Configuration

```yaml
bmp:
  listen: ":11019"

rtr:
  caches:
    - address: "localhost:3323"
```

This is enough to get RAVEN running — BMP listener on port 11019, RTR
connecting to a local Routinator instance.

## Full Configuration Reference

```yaml
# raven.yaml

bmp:
  listen: ":11019"                  # BMP listener address (default: :11019)

rtr:
  caches:
    - address: "rpki1.example.com:3323"
      preference: 1                 # Lower = higher preference
      transport: tcp                # tcp | tls
    - address: "rpki2.example.com:3323"
      preference: 2
      transport: tls
      tls:
        ca: "/etc/raven/rpki-ca.pem"
        tls-min-version: "1.2"      # 1.2 (default) | 1.3
  rtr-version: auto                 # auto | 1 | 2 (default: auto)
  expire-interval: 7200             # Override cache expire in seconds

validation:
  rov: true                         # Enable ROV annotation (default: true)
  aspa: true                        # Enable ASPA annotation (default: true)
  aspa-default-procedure: upstream  # upstream | downstream | auto
  aspa-as-set-behavior: unverifiable # unverifiable | best-effort

outputs:
  prometheus:
    listen: ":9595"                 # Prometheus metrics endpoint
    path: "/metrics"
  otel:                             # optional OTLP export (see below)
    enabled: false
  # kafka:                          # optional event sink
  #   brokers: ["localhost:9092"]
  #   topic: "raven-events"
  # file:                           # optional event sink
  #   path: "/var/log/raven/events.ndjson"

logging:
  level: info                       # debug | info | warn | error
  format: text                      # text | json
```

!!! note
    There is no `api:` section. The API server always binds to `:11020`
    (not configurable). See [API Section](#api-section) below.

## Configuration Options Explained

### BMP Section

**`bmp.listen`** — The address and port RAVEN listens on for incoming BMP
sessions from routers. Use `0.0.0.0:11019` to listen on all interfaces,
or `192.0.2.10:11019` to bind to a specific interface.

!!! warning
    The BMP listener is unauthenticated TCP. It should be reachable only from
    your routers — not from the public internet. Use firewall rules or bind to
    a management interface.

### RTR Section

**`rtr.caches`** — List of RPKI validators to connect to. RAVEN connects to
all configured caches and uses the one with the lowest preference number as
primary. If the primary fails, it falls back to the next.

**`rtr.rtr-version`** — Protocol version negotiation:

- `auto` (default) — RAVEN negotiates the highest version the validator supports.
  RTR v2 gives ASPA data; RTR v1 gives VRPs only.
- `1` — Force RTR v1 (VRPs only, no ASPA)
- `2` — Force RTR v2 (VRPs + ASPA). Will fail if validator doesn't support it.

**`rtr.expire-interval`** — Override the cache expiry interval in seconds.
Use this if your validator's expire interval is too short for your environment.

### Validation Section

**`validation.aspa-default-procedure`** — How to determine the ASPA verification
procedure for each route:

- `upstream` — Always use upstream verification (catches route leaks from customers/peers)
- `downstream` — Always use downstream verification (weaker, use when receiving from providers)
- `auto` — Infer from BGP Role capability (RFC 9234) in BMP peer data. Falls back
  to `upstream` if BGP Role is not available.

**`validation.aspa-as-set-behavior`** — How to handle AS_PATHs containing AS_SET segments:

- `unverifiable` (default) — Mark the route as ASPA Unverifiable. Safe and correct.
- `best-effort` — Validate AS_SEQUENCE segments, skip AS_SET segments.

### Outputs Section

**`outputs.prometheus`** — Exposes a Prometheus `/metrics` endpoint. Point your
Prometheus scraper at this address. See [Observability & Metrics](observability.md)
for the full metrics reference.

**`outputs.otel`** — Optional OTLP metrics export alongside Prometheus. See
[OpenTelemetry Configuration](#opentelemetry-configuration) below.

**`outputs.kafka`** — Optional event sink that publishes events to a Kafka
topic (`brokers`, `topic`).

**`outputs.file`** — Optional event sink that appends events to a file
(`path`, `rotation`).

### API Section

The API server is **not configurable**. It always binds to `:11020`
(hardcoded — see `internal/server/server.go:39` and `:339`). There is no
`api:` section in `raven.yaml`; adding one has no effect.

The CLI's `--address` flag (default `localhost:11020`) is a **client-side**
setting: it tells the CLI where to reach a running RAVEN, but does not
configure the server's bind address. To query a remote instance, pass
`--address <host>:11020` on each command (see [Pointing the CLI at a Remote
RAVEN Instance](#pointing-the-cli-at-a-remote-raven-instance) below).

## Multiple RTR Caches

Running two RTR caches provides redundancy:

```yaml
rtr:
  caches:
    - address: "routinator1.example.com:3323"
      preference: 1
    - address: "routinator2.example.com:3323"
      preference: 2
```

RAVEN connects to both but treats preference 1 as authoritative. If it goes
down, RAVEN automatically uses preference 2 and logs a warning.

## Transport Security

RAVEN supports TLS on both its outbound RTR connection
(to your RPKI validator) and its inbound BMP listener
(from your routers).

### RTR over TLS

To encrypt the RTR session between RAVEN and your RPKI
validator, set `transport: tls` on the cache entry and
optionally provide a CA certificate to verify the
validator's identity:

```yaml
rtr:
  caches:
    - address: "rpki1.example.com:3324"
      preference: 1
      transport: tls
      tls:
        ca: "/etc/raven/rpki-ca.crt"   # verify validator cert
        # cert: "/etc/raven/client.crt" # optional: mutual TLS
        # key:  "/etc/raven/client.key"
        # tls-min-version: "1.2"        # 1.2 (default) | 1.3
```

If `ca` is omitted, RAVEN uses the system root CA pool.
Mutual TLS (client certificate) is supported by adding
`cert` and `key`. `tls-min-version` pins the minimum
negotiated TLS version — `"1.2"` (default) or `"1.3"` —
and is available on both the RTR `tls:` block above and
the BMP `tls:` block below.

**Routinator TLS setup:**

Routinator supports RTR-over-TLS natively. Start it
with the `--rtr-tls` flag (note: separate from
`--rtr-listen`):

```bash
routinator server \
  --rtr-tls 127.0.0.1:3324 \
  --rtr-tls-cert /path/to/server.crt \
  --rtr-tls-key  /path/to/server.key
```

The IANA-assigned port for RTR-over-TLS is 324; use
3324 to avoid requiring root privileges.

**RTR version pinning:**

If your validator does not support RTR v2, pin to v1
to avoid reconnection loops during version negotiation:

```yaml
rtr:
  rtr-version: "1"
  caches:
    - address: "rpki1.example.com:3324"
      transport: tls
      tls:
        ca: "/etc/raven/rpki-ca.crt"
```

### BMP Listener TLS

To require TLS on incoming BMP connections from your
routers, add a `tls:` block under `bmp:`. A server
certificate and key are required; a CA is optional and
enables mutual TLS (verifying router client certs):

```yaml
bmp:
  listen: ":11019"
  tls:
    cert: "/etc/raven/bmp.crt"   # required
    key:  "/etc/raven/bmp.key"   # required
    ca:   "/etc/raven/ca.crt"    # optional: mutual TLS
    # tls-min-version: "1.2"     # 1.2 (default) | 1.3
```

Without the `tls:` block, the BMP listener accepts
plain TCP connections (default, compatible with all
BMP-capable routers including FRR and SR Linux).

> **Note:** Most routers do not currently support
> BMP-over-TLS. This option is intended for deployments
> where RAVEN is reachable across untrusted network
> segments. For lab and internal deployments, plain TCP
> is standard.

### Generating Test Certificates

For lab use, generate a self-signed CA and server cert
with v3 extensions (required by modern TLS stacks):

```bash
# CA
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:P-256 \
  -keyout ca.key -out ca.crt -days 365 -nodes \
  -subj "/CN=raven-ca" \
  -addext "basicConstraints=critical,CA:true" \
  -addext "keyUsage=critical,keyCertSign,cRLSign"

# Server cert (for Routinator or BMP listener)
openssl req -newkey ec -pkeyopt ec_paramgen_curve:P-256 \
  -keyout server.key -out server.csr -nodes \
  -subj "/CN=localhost"

openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out server.crt -days 365 \
  -extfile <(printf "subjectAltName=DNS:localhost,IP:127.0.0.1\n\
basicConstraints=CA:false\n")
```

## Environment Variable Overrides

Any config value can be overridden with an environment variable using the
`RAVEN_` prefix and `_` as separator:

```bash
RAVEN_BMP_LISTEN=":11019" raven serve
RAVEN_LOGGING_LEVEL=debug raven serve
```

## Pointing the CLI at a Remote RAVEN Instance

By default the CLI connects to `localhost:11020`. To query a remote instance,
pass `--address` on each command:

```bash
raven routes --address 192.0.2.10:11020 --posture origin-invalid
```

`--address` is a per-invocation flag — it is not read from `raven.yaml` or from
an environment variable. To avoid repeating it, wrap it in a shell alias:

```bash
alias raven-prod='raven --address 192.0.2.10:11020'
```

---

## Events Configuration

The `events` section configures the Event Engine — rules that trigger actions
when route postures change.

```yaml
events:
  rules:
    - name: "alert-origin-invalid"
      trigger:
        type: posture_change
        postures: ["origin-invalid"]
      cooldown: 60s
      actions:
        - type: log
          level: warn
        - type: webhook
          url: "https://hooks.example.com/raven"
          secret: "your-hmac-secret"
          max_attempts: 3
          timeout: 5s
        - type: flowspec
          gobgp_address: "localhost:50051"
          dry_run: true
          ttl: 5m
          rule_action: drop
          max_rules: 100
          reaper_interval: 30s

    - name: "alert-route-leak"
      trigger:
        type: posture_change
        postures: ["path-suspect"]
      cooldown: 60s
      actions:
        - type: log
          level: warn
        - type: webhook
          url: "https://hooks.example.com/raven"

    - name: "alert-any-route-via-as64496"
      trigger:
        type: asn
        asns: [64496]
        asn_match: path       # "origin" (default) or "path"
      cooldown: 60s
      actions:
        - type: log
          level: info

    - name: "alert-hijack-of-my-prefixes"
      trigger:
        type: protected_asn
        asns: [64511]
      cooldown: 60s
      actions:
        - type: log
          level: warn
        - type: webhook
          url: "https://hooks.example.com/raven"
```

**Trigger types:**

| Type | Description |
|---|---|
| `posture` | Fires whenever a route currently has one of the listed postures |
| `posture_change` | Fires when a route's posture *changes* to one of the listed values |
| `prefix` | Fires on routes matching a specific prefix |
| `asn` | Fires when a route involves one of the listed ASNs. `asn_match: origin` (default) matches the origin only; `asn_match: path` matches any ASN in the full path |
| `protected_asn` | Fires when an origin-invalid route hijacks a prefix authorised for one of the listed ASNs |
| `cache_unhealthy` | Fires when an RTR cache becomes unreachable |
| `compound` | Combines sub-`triggers` with an `operator` of `and` or `or` |

**Action types:**

| Type | Description |
|---|---|
| `log` | Log the event at the specified level |
| `webhook` | HTTP POST with JSON payload, HMAC-SHA256 signed |
| `flowspec` | Generate a Flowspec drop rule, inject via GoBGP |

**Cooldown** — Minimum time between repeated firings of the same rule for the
same prefix+peer combination. Prevents alert storms during flapping.

**Webhook payload** — The JSON body sent to your webhook URL:

```json
{
  "id": "b65b80a6-...",
  "timestamp": "2026-05-06T14:23:01Z",
  "type": "posture_change",
  "rule_name": "alert-origin-invalid",
  "prefix": "192.0.2.0/24",
  "peer_addr": "10.0.0.1",
  "peer_asn": 65000,
  "origin_asn": 64496,
  "protected_asns": [64511],
  "old_posture": "origin-only",
  "new_posture": "origin-invalid",
  "rov_state": "Invalid",
  "aspa_state": "Unknown",
  "router_id": "10.0.0.1"
}
```

`rule_name` identifies which rule fired. `protected_asns` lists the ASNs named as
authorized originators in matched VRPs — present only when there are covering VRPs
(relevant for `protected_asn` trigger alerts).

**Flowspec options:**

| Option | Default | Description |
|---|---|---|
| `gobgp_address` | `localhost:50051` | GoBGP gRPC management endpoint |
| `dry_run` | `true` | Log rules without injecting into GoBGP |
| `ttl` | `1h` | Per-rule lifetime before auto-expiry |
| `rule_action` | `drop` | Flowspec action: `drop` or `rate-limit` |
| `max_rules` | `100` | Maximum concurrently active rules |
| `reaper_interval` | `30s` | How often to check for and expire TTL'd rules |
| `approval_webhook` | — | URL to POST proposed rules to before injecting |

!!! tip
    Always start with `dry_run: true`. Use `raven flowspec list` to review
    generated rules, then toggle specific rules live with
    `raven flowspec toggle "<key>"` when you are confident they are correct.

---

## Persistence (Warm-Start Snapshots)

The `persistence` section configures warm-start snapshots — saving the route
table and RPKI cache to disk so RAVEN can restore state quickly on restart.

```yaml
persistence:
  enabled: true
  snapshot_dir: "/var/lib/raven/snapshots"   # directory, not a file
  stale_eviction_timeout: 5m                 # evict unconfirmed restored routes after this
```

**Options:**

| Option | Default | Description |
|---|---|---|
| `enabled` | `false` | Enable snapshot save/restore |
| `snapshot_dir` | `~/.raven/snapshots` | Directory for snapshot files (supports `~` expansion) |
| `stale_eviction_timeout` | `5m` | Routes restored from a snapshot but never confirmed by a live BMP message are evicted this long after startup |

Without persistence, RAVEN rebuilds its route table from BMP table dumps on
restart (typically 30–90 seconds). With it, the route table is available
immediately on startup while BMP reconnects in the background.

---

## OpenTelemetry Configuration

The `outputs.otel` section configures OTLP metrics export alongside Prometheus.

```yaml
outputs:
  otel:
    enabled: true
    endpoint: "otel-collector:4317"
    protocol: grpc               # grpc | http
    interval: 30s
    insecure: true              # true = no TLS on the OTLP connection
    headers:                    # optional per-request headers (e.g. auth)
      x-api-key: "secret"
    resource_attributes:        # attached to every exported metric
      service.name: "raven"
      service.version: "v0.1.0"
```

**Options:**

| Option | Default | Description |
|---|---|---|
| `enabled` | `false` | Enable OpenTelemetry export |
| `endpoint` | `localhost:4317` | OTLP collector endpoint |
| `protocol` | `grpc` | Export protocol: `grpc` or `http` |
| `interval` | `30s` | Metrics export interval |
| `insecure` | `true` | Disable TLS on the OTLP connection |
| `headers` | — | Optional map of headers sent with each OTLP request |
| `resource_attributes` | `service.name=raven`, `service.version=v0.1.0` | Resource attributes attached to exported metrics. Set the service name here — there is no separate `service_name` option |

RAVEN exports the same metrics via OTLP as via Prometheus. Both outputs
can be active simultaneously. RAVEN continues normally if the OTLP
collector is unreachable — export failures are logged but not fatal.
