# RAVEN
**Ravens see what you can't.**

*Documentation for RAVEN (bgp-routing-security-monitor) — BMP + RPKI ROV + ASPA path validation in a single binary.*

---

RAVEN is an open-source, lightweight, single-binary routing security observability tool. It connects directly to your routers via BMP (BGP Monitoring Protocol) and your RPKI validators via RTR, annotates every route with its security posture in real-time, and exposes the results through a CLI, Prometheus metrics, and Grafana dashboards.

## What Does RAVEN Answer?

> *"Show me every route I'm receiving, whether the origin is RPKI-valid, whether the AS_PATH is ASPA-valid, and what I should do about the ones that aren't."*

No existing tool answers this question. BMP collectors don't validate. RPKI validators don't see live routes. External monitors can't see your internal routing state. RAVEN fills that gap.

## Quick Start

```bash
# Install
go install github.com/nokia/bgp-routing-security-monitor/cmd/raven@latest

# Start RAVEN — point at your router (BMP) and validator (RTR)
raven serve --config raven.yaml

# In another terminal — see your routes
raven routes --posture origin-invalid
```

## Key Capabilities

| Capability | Description |
|---|---|
| **BMP Ingest** | Accept BMP sessions from any vendor's router |
| **IPv6** | IPv6 route monitoring via BMP `MP_REACH_NLRI` / `MP_UNREACH_NLRI`, with ROV validation against IPv6 ROAs |
| **ROV** | Route Origin Validation per RFC 6811 |
| **ASPA** | AS_PATH validation per draft-ietf-sidrops-aspa-verification-24 |
| **Combined Posture** | Unified security posture per route (Secured / Path-Suspect / Origin-Invalid / ...) |
| **What-If** | Simulate impact of deploying reject-invalid or ASPA enforcement |
| **ASPA Recommender** | Suggest ASPA objects based on observed AS_PATHs |
| **Event Engine** | Trigger webhooks and Flowspec rules on posture changes |
| **Flowspec** | Automated mitigation — detect, generate, inject, expire via GoBGP |
| **raven audit** | Read-only security posture report per router |
| **Warm-start** | Snapshot and restore route table across restarts |
| **Prometheus** | `/metrics` endpoint for Grafana dashboards |
| **OpenTelemetry** | OTLP metrics export alongside Prometheus |
| **Single Binary** | Zero dependencies — download and run |

## Why Now?

ASPA has crossed into production availability — ARIN enabled ASPA object creation in January 2026, RIPE NCC in December 2025. Adoption is under 1% of the global ASN space. The tooling gap is a significant barrier. RAVEN is the first operational tool that brings ASPA validation to your live routing table.

## Get Started

- [Installation](getting-started/install.md) — get the binary
- [Quick Start](getting-started/quick-start.md) — first annotated route in minutes
- [Concepts](getting-started/concepts.md) — BMP, ROV, ASPA explained
- [Demo Lab](lab/overview.md) — run the full stack on your laptop
- [Active Response](user-guide/active-response.md) — webhooks, Flowspec, event engine

## Project

RAVEN is open source under the BSD-3-Clause license, developed at Nokia.

- **GitHub:** [nokia/bgp-routing-security-monitor](https://github.com/nokia/bgp-routing-security-monitor)
- **License:** BSD-3-Clause
- **Standards:** RFC 7854, RFC 6811, RFC 8210, draft-ietf-sidrops-aspa-verification-24
