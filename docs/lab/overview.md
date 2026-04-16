# Demo Lab Overview

RAVEN ships with a complete Containerlab topology that runs the full routing
security stack on your laptop. This is the same lab used for conference demos.

## What the Lab Gives You

- Three FRR routers simulating a real network topology
- Live BGP sessions with BMP streaming to RAVEN
- Routinator serving real RPKI data including ASPA objects
- Prometheus and Grafana with pre-built dashboards
- Scripted attack scenarios — origin hijack, route leak, what-if simulation

## Topology
                Internet (AS2121)
                193.0.0.0/21
                     │
                     │ eBGP
                     ▼
                Upstream (AS65000)
                10.0.0.0/8
                     │
                     │ eBGP + BMP → RAVEN
                     ▼
                Edge (AS65001)
                172.16.0.0/12

- **Internet (AS2121)** — simulates the upstream internet, announces
  `193.0.0.0/21`. Used as the attacker in hijack scenarios.
- **Upstream (AS65000)** — your upstream provider, announces `10.0.0.0/8`.
  Sends BMP to RAVEN.
- **Edge (AS65001)** — your edge router, announces `172.16.0.0/12`.
  Sends BMP to RAVEN.

RAVEN sees everything the edge router receives from upstream, and everything
upstream receives from the internet.

## Prerequisites

- **Containerlab** — [install guide](https://containerlab.dev/install/)
- **Docker** — running and accessible without sudo (or use sudo)
- **Routinator** — installed as a native binary (not Docker)
- **RAVEN** — built from source or installed via go install
- **Prometheus** — Docker container
- **Grafana** — Docker container

## Installing Routinator

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
