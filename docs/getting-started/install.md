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
[GitHub releases page](https://github.com/nokia-steward/bgp-routing-security-monitor/releases).

```bash
# Linux amd64
curl -Lo raven https://github.com/nokia-steward/bgp-routing-security-monitor/releases/latest/download/raven-linux-amd64
chmod +x raven
sudo mv raven /usr/local/bin/
```

```bash
# Linux arm64
curl -Lo raven https://github.com/nokia-steward/bgp-routing-security-monitor/releases/latest/download/raven-linux-arm64
chmod +x raven
sudo mv raven /usr/local/bin/
```

```bash
# macOS amd64
curl -Lo raven https://github.com/nokia-steward/bgp-routing-security-monitor/releases/latest/download/raven-darwin-amd64
chmod +x raven
sudo mv raven /usr/local/bin/
```

```bash
# macOS arm64 (Apple Silicon)
curl -Lo raven https://github.com/nokia-steward/bgp-routing-security-monitor/releases/latest/download/raven-darwin-arm64
chmod +x raven
sudo mv raven /usr/local/bin/
```

## Build from Source

```bash
git clone https://github.com/nokia-steward/bgp-routing-security-monitor.git
cd bgp-routing-security-monitor
go build -o raven ./cmd/raven
sudo mv raven /usr/local/bin/
```

## Requirements

| Requirement | Details |
|---|---|
| **Routers** | Any BMP-capable router — SR Linux, SR OS, Junos, IOS-XR, Arista EOS, FRR |
| **RPKI Validator** | Any RTR-speaking validator — Routinator, rpki-client, FORT |
| **ASPA** | Routinator 0.16 or later (ASPA on by default), or Routinator 0.15.x and older started with `--enable-aspa`, for RTR v2 + ASPA PDU support |
| **Ports** | BMP listener: `11019/tcp`, API: `11020/tcp`, Prometheus: `9595/tcp` |

## RPKI Validator Setup

RAVEN consumes VRP and ASPA data from your existing RPKI validator via RTR.
It does not replace your validator — it connects to it.

For ASPA support, you need **Routinator** serving ASPA over RTR v2. How you
turn ASPA on depends on the Routinator version (`routinator --version`).

Routinator 0.16 and later process ASPA by default, so you set nothing:

```bash
routinator server --rtr 127.0.0.1:3323
```

`--disable-aspa`, or `enable-aspa = false` in the config file, turns ASPA
off. Do not use either for RAVEN.

Routinator 0.15.x and older need `--enable-aspa` before the `server`
command, or `enable-aspa = true` in the config file:

```bash
routinator --enable-aspa server --rtr 127.0.0.1:3323
```

Routinator 0.16 rejects `--enable-aspa` as an unknown option. It still
accepts `enable-aspa = true` in the config file, so a config file with that
line works with both versions.

!!! note
    Routinator takes approximately 4 minutes for a cold start while it syncs
    the RPKI repository. Subsequent starts use a warm cache and take around
    13 seconds. Do not restart Routinator unnecessarily.

## Verify Installation

```bash
raven version
```

Expected output: raven version v0.1.0 (commit: abc1234, built: 2026-04-15T00:00:00Z)
