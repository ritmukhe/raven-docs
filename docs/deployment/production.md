# Production Setup

This guide covers deploying RAVEN in a production environment against real
routers and a production RPKI validator.

## Architecture
```
┌─────────────────┐     BMP      ┌─────────────────────┐
│  Edge Router 1  │─────────────▶│                     │
├─────────────────┤              │       RAVEN          │
│  Edge Router 2  │─────────────▶│   BMP  :11019        │
└─────────────────┘              │   API  :11020        │
│   Prom :9595         │
┌─────────────────┐     RTR      │                     │
│   Routinator    │◀─────────────│                     │
└─────────────────┘              └──────────┬──────────┘
│ /metrics
┌──────────▼──────────┐
│  Prometheus+Grafana  │
└─────────────────────┘
```

## Step 1 — Install RAVEN

On the monitoring host:

```bash
go install github.com/nokia/bgp-routing-security-monitor/cmd/raven@latest
```

Or download a pre-built binary from the
[releases page](https://github.com/nokia/bgp-routing-security-monitor/releases)
and place it in `/usr/local/bin/`.

## Step 2 — Create the Config File

```bash
sudo mkdir -p /etc/raven
sudo cat > /etc/raven/raven.yaml << 'EOF'
bmp:
  listen: "0.0.0.0:11019"

rtr:
  caches:
    - address: "rpki-validator.example.com:3323"
      preference: 1
    - address: "rpki-validator-backup.example.com:3323"
      preference: 2
  rtr-version: auto

validation:
  rov: true
  aspa: true
  aspa-default-procedure: upstream

outputs:
  prometheus:
    listen: "0.0.0.0:9595"
    path: "/metrics"

api:
  listen: "0.0.0.0:11020"

logging:
  level: info
  format: json
