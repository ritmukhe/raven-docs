# Installation

RAVEN is a single binary with no runtime dependencies. Pick the method that suits you.

## Go Install

If you have Go 1.24+ installed:

```bash
go install github.com/nokia/bgp-routing-security-monitor/cmd/raven@latest
```

Verify:

```bash
raven version
```

## Pre-built Binaries

Download the latest binary for your platform from the
[GitHub releases page](https://github.com/nokia/bgp-routing-security-monitor/releases).

```bash
# Linux amd64
curl -Lo raven https://github.com/nokia/bgp-routing-security-monitor/releases/latest/download/raven-linux-amd64
chmod +x raven
sudo mv raven /usr/local/bin/
```

```bash
# Linux arm64
curl -Lo raven https://github.com/nokia/bgp-routing-security-monitor/releases/latest/download/raven-linux-arm64
chmod +x raven
sudo mv raven /usr/local/bin/
```

```bash
# macOS amd64
curl -Lo raven https://github.com/nokia/bgp-routing-security-monitor/releases/latest/download/raven-darwin-amd64
chmod +x raven
sudo mv raven /usr/local/bin/
```

```bash
# macOS arm64 (Apple Silicon)
curl -Lo raven https://github.com/nokia/bgp-routing-security-monitor/releases/latest/download/raven-darwin-arm64
chmod +x raven
sudo mv raven /usr/local/bin/
```

## Build from Source

```bash
git clone https://github.com/nokia/bgp-routing-security-monitor.git
cd bgp-routing-security-monitor
go build -o raven ./cmd/raven
sudo mv raven /usr/local/bin/
```

## Requirements

| Requirement | Details |
|---|---|
| **Routers** | Any BMP-capable router — SR Linux, SR OS, Junos, IOS-XR, Arista EOS, FRR |
| **RPKI Validator** | Any RTR-speaking validator — Routinator, rpki-client, FORT |
| **ASPA** | Routinator with `--enable-aspa` for RTR v2 + ASPA PDU support |
| **Ports** | BMP listener: `11019/tcp`, API: `11020/tcp`, Prometheus: `9595/tcp` |

## RPKI Validator Setup

RAVEN consumes VRP and ASPA data from your existing RPKI validator via RTR.
It does not replace your validator — it connects to it.

For ASPA support, you need **Routinator** with RTR v2 enabled:

```bash
routinator server --enable-aspa --rtr 127.0.0.1:3323
```

### Docker

A Docker image is available at `nlnetlabs/routinator:latest`:

```bash
docker run -d --name routinator \
  -p 3323:3323 \
  nlnetlabs/routinator:latest \
  server --enable-aspa --rtr 0.0.0.0:3323
```

Point RAVEN at `localhost:3323` in your RTR config.

!!! note
    Routinator takes approximately 4 minutes for a cold start while it syncs
    the RPKI repository. Subsequent starts use a warm cache and take around
    13 seconds. Do not restart Routinator unnecessarily.

## Verify Installation

```bash
raven version
```

Expected output: raven version v0.1.0 (commit: abc1234, built: 2026-04-15T00:00:00Z)
