# Quick Start

This guide gets you from zero to seeing your first annotated routes in minutes.

## What You Need

- RAVEN binary installed (see [Installation](install.md))
- At least one BMP-capable router
- An RPKI validator running RTR (Routinator recommended)

## Step 1 — Create a Config File

Create `raven.yaml` in your working directory:

```yaml
bmp:
  listen: ":11019"

rtr:
  caches:
    - address: "localhost:3323"
      preference: 1

validation:
  rov: true
  aspa: true

outputs:
  prometheus:
    listen: ":9595"
    path: "/metrics"

logging:
  level: info
  format: text
```

## Step 2 — Start RAVEN

```bash
raven serve --config raven.yaml
```

You should see:
INFO BMP listener started address=:11019
INFO RTR client connecting cache=localhost:3323
INFO RTR session established version=2 vrps=542000 aspas=1407
INFO Prometheus metrics available address=:9595

## Step 3 — Configure Your Router

Point your router's BMP configuration at RAVEN's BMP listener.

**FRR example:**
router bgp 65000
bmp targets raven
address 192.0.2.10 port 11019
monitor ipv4 unicast pre-policy
monitor ipv6 unicast pre-policy
exit
exit

**SR Linux example:**
network-instance default {
protocols {
bgp {
bmp {
station raven {
connection {
destination-address 192.0.2.10
destination-port 11019
}
local-address 0.0.0.0
}
}
}
}
}

## Step 4 — Check Peer Status

Once your router connects:

```bash
raven status
```
BMP PEERS
192.0.2.1  AS65001  state=UP  routes=842341  since=2m ago
RTR CACHES
localhost:3323  state=UP  serial=4821  vrps=542000  aspas=1407  last-sync=45s ago

## Step 5 — Query Your Routes

```bash
# Show all routes with their security posture
raven routes

# Show only routes failing ROV
raven routes --posture origin-invalid

# Show routes with ASPA path violations
raven routes --posture path-suspect

# Show routes for a specific prefix
raven routes --prefix 1.0.0.0/24
```

## Step 6 — Watch for Problems in Real Time

```bash
raven watch --posture origin-invalid
```

This streams any new origin-invalid routes to your terminal as they arrive.
Leave it running — it will catch hijacks and ROV state changes as they happen.

## Step 7 — Run a What-If Simulation

```bash
raven what-if --reject-invalid
```

This shows you how many routes would be dropped if you deployed reject-invalid
policy today — without changing anything on your routers.

## Next Steps

- [Configuration](../user-guide/configuration.md) — full config file reference
- [CLI Reference](../user-guide/cli-reference.md) — every command and flag
- [Security Postures](../user-guide/security-postures.md) — what each posture means and what to do
- [Demo Lab](../lab/overview.md) — run attack scenarios on your laptop
