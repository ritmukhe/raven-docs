# Demo Lab Overview

RAVEN ships with a complete Containerlab topology that runs the full routing
security stack on your laptop. This is the same lab used for conference demos.

## What the Lab Gives You

- Four FRR routers simulating a real network topology (including an attacker node)
- Live BGP sessions with BMP streaming to RAVEN
- Routinator serving real RPKI data including ASPA objects
- Prometheus and Grafana with pre-built dashboards
- Scripted attack scenarios — origin hijack, route leak, what-if simulation

## Topology

The lab topology consists of four FRR routers and RAVEN connected via
Containerlab:
```
                Internet (AS2121)
                       │ eBGP
                       ▼
        Upstream (AS65000) ─── eBGP ─── Attacker (AS65099)
                       │ eBGP                           │ eBGP
                       ▼                                │
                Edge (AS65001) ──────────────────────────┘
                       │ BMP (pre-policy)
                       ▼
                     RAVEN ◄── RTR ── Routinator
                       │
                       ▼
              Prometheus + Grafana
```

| Node | ASN | Role |
|---|---|---|
| internet | AS2121 | Originates `193.0.0.0/21` (RIPE NCC prefix, real ASPA record) |
| upstream | AS65000 | Transit provider, sends post-policy BMP to RAVEN |
| edge | AS65001 | Edge router, sends pre-policy BMP to RAVEN |
| attacker | AS65099 | Peered with upstream and edge, announces nothing at rest |

The attacker node is used for origin hijack (Scenario 2), stealthy hijack
(Scenario 3), and the route leak uses the internet router's real ASPA record
(AS2121, providers: AS3333 only) to produce a genuine ASPA:Invalid result.

## Prerequisites

- **Containerlab** — [install guide](https://containerlab.dev/install/)
- **Docker** — running and accessible without sudo (or use sudo)
- **Routinator** — native binary or Docker (see Installing Routinator below)
- **RAVEN** — built from source or installed via go install
- **Prometheus** — Docker container
- **Grafana** — Docker container

## Installing Routinator

### Docker

A Docker image is available at `nlnetlabs/routinator:latest`. Save the following as `~/.routinator.conf`:

```toml
repository-dir = "/home/routinator/.rpki-cache/repository"
rtr-listen = ["0.0.0.0:3323"]
http-listen = ["0.0.0.0:8323"]
disable-rsync = true
enable-aspa = true
```

Then start the container:

```bash
docker run -d --name routinator \
  -p 3323:3323 \
  -v ~/.routinator.conf:/home/routinator/.routinator.conf:ro \
  -v ~/.rpki-cache:/home/routinator/.rpki-cache \
  nlnetlabs/routinator:latest \
  server
```

### Native Binary

```bash
# Install via cargo
cargo install routinator

# Or download a pre-built binary from
# https://github.com/NLnetLabs/routinator/releases

# Create config
mkdir -p ~/.rpki-cache/repository
cat > ~/.routinator.conf << 'CONF'
repository-dir = "/home/YOUR_USER/.rpki-cache/repository"
rtr-listen = ["127.0.0.1:3323"]
http-listen = ["127.0.0.1:8323"]
disable-rsync = true
CONF

# Start Routinator (first run takes ~4 minutes to sync)
routinator server &
```

!!! warning
    Routinator takes approximately 4 minutes for a cold start while it
    downloads and validates the RPKI repository. A warm cache restart takes
    about 13 seconds. **Do not kill Routinator between demos** — the warm
    cache is essential for reliable demo startup.

## Quick Lab Start

From the RAVEN repository root:

```bash
bash lab/demo-master.sh setup
```

This command:

1. Deploys the Containerlab topology
2. Waits for FRR BGP sessions to converge
3. Verifies Routinator is running and synced
4. Starts RAVEN
5. Starts Prometheus and Grafana
6. Imports the Grafana dashboard
7. Runs a baseline check

When setup completes you will see:
✓ Containerlab topology running
✓ BGP sessions converged
✓ Routinator ready (vrps=542000 aspas=1407)
✓ RAVEN running
✓ Prometheus running
✓ Grafana running at http://localhost:3000
✓ Baseline: 0 origin-invalid, 0 path-suspect
Demo is ready.
Grafana: http://localhost:3000  (admin/admin)

## Demo Commands Reference

```bash
bash lab/demo-master.sh setup        # Start everything
bash lab/demo-master.sh down         # Stop everything cleanly
bash lab/demo-master.sh baseline     # Show clean route table
bash lab/demo-master.sh hijack       # Inject origin hijack
bash lab/demo-master.sh hijack-clean # Withdraw the hijack
bash lab/demo-master.sh leak         # Inject route leak
bash lab/demo-master.sh leak-clean   # Withdraw the leak
bash lab/demo-master.sh whatif       # Run what-if simulator
bash lab/demo-master.sh recommend    # Run ASPA recommender
```

## Grafana Dashboard

Open [http://localhost:3000](http://localhost:3000) in your browser.

- Username: `admin`
- Password: `admin`

The Security Posture Overview dashboard updates in real time as you run
the attack scenarios.

## Stopping the Lab

```bash
bash lab/demo-master.sh down
```

This stops RAVEN, Prometheus, Grafana, and destroys the Containerlab topology
cleanly. Always use this rather than killing processes manually.
