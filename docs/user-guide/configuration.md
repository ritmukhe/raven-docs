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

api:
  listen: "localhost:11020"         # REST/gRPC API (default: localhost:11020)

logging:
  level: info                       # debug | info | warn | error
  format: text                      # text | json
```

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
Prometheus scraper at this address. See [Prometheus & Grafana](observability.md)
for the full metrics reference.

### API Section

**`api.listen`** — The address the REST/gRPC API listens on. The CLI commands
connect to this address. Default is `localhost:11020` — change this if you want
to query RAVEN from another machine.

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

## Environment Variable Overrides

Any config value can be overridden with an environment variable using the
`RAVEN_` prefix and `_` as separator:

```bash
RAVEN_BMP_LISTEN=":11019" raven serve
RAVEN_LOGGING_LEVEL=debug raven serve
```

## Pointing the CLI at a Remote RAVEN Instance

By default the CLI connects to `localhost:11020`. To query a remote instance:

```bash
raven routes --address 192.0.2.10:11020 --posture origin-invalid
```

Or set it permanently:

```yaml
# raven.yaml (client side)
api:
  listen: "192.0.2.10:11020"
```
