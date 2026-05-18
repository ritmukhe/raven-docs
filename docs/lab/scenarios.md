# Attack Scenarios

The demo lab includes scripted scenarios that demonstrate RAVEN's detection
capabilities in real time. Run these after `demo-master.sh setup` completes.

## Running the Full Conference Demo

To run all scenarios in sequence with narrated pauses:

```bash
./demo-master.sh lacnic
```

This runs the complete 6-scenario LACNIC demo sequence: baseline, origin
hijack, stealthy hijack, route leak, RTR cache failure, and audit report.

## Baseline

Before running any scenario, verify the baseline is clean:

```bash
bash lab/demo-master.sh baseline
```

Expected output:
BASELINE CHECK
origin-invalid routes: 0
path-suspect routes:   0
secured routes:        6
origin-only routes:    4
Baseline is clean. Ready for demo.

Open Grafana at [http://localhost:3000](http://localhost:3000) and keep it
visible alongside your terminal. You will see the panels update in real time
as each scenario runs.

Also open a second terminal and run:

```bash
raven watch
```

This streams all validation state changes live.

---

## Scenario 1 — Origin Hijack

An attacker announces your prefix from an unauthorized ASN. ROV catches this
because a ROA exists authorizing only the legitimate origin ASN.

**Inject the hijack:**

```bash
bash lab/demo-master.sh hijack
```

What happens:

- The internet router (AS2121) announces `192.0.2.0/24` — a prefix it does
  not own and has no ROA for
- RAVEN receives this via BMP from the upstream router
- ROV lookup finds a ROA authorizing a different ASN
- RAVEN marks the route `origin-invalid`

**What you will see in `raven watch`:**
[14:23:01] NEW  192.0.2.0/24  via 10.0.0.1  AS2121  ROV:Invalid  posture:origin-invalid
Matched VRP: ROA{192.0.2.0/24, AS65000, maxLength=/24}
Reason: Origin AS2121 not authorized — ROA authorizes AS65000

**What you will see in Grafana:**

- `origin-invalid` counter jumps from 0 to 1
- The origin-invalid routes table shows the hijacked prefix

**CLI investigation:**

```bash
raven routes --posture origin-invalid
```
PREFIX          PEER          ORIGIN  ROV      ASPA     POSTURE
192.0.2.0/24    10.0.0.1   AS2121  Invalid  Unknown  origin-invalid

**Clean up:**

```bash
bash lab/demo-master.sh hijack-clean
```

The route is withdrawn. `raven watch` shows the route disappearing.
Grafana returns to baseline.

---

## Scenario 2 — Route Leak

A route leak occurs when an AS re-announces routes in the wrong direction.
ROV cannot catch this — the origin ASN is legitimate. ASPA catches it because
the AS_PATH contains a hop that violates published provider relationships.

**Inject the route leak:**

```bash
bash lab/demo-master.sh leak
```

What happens:

- The upstream router (AS65000) leaks a route it received from the internet
  back to the edge router via an unauthorized path
- The origin AS1199 (SURFnet) has a valid ROA — so ROV passes
- RAVEN walks the AS_PATH and finds a hop that violates an ASPA record
- RAVEN marks the route `path-suspect`

**What you will see in `raven watch`:**
[14:24:15] NEW  145.102.136.0/22  via 10.0.0.1  AS1199  ROV:Valid  ASPA:Invalid  posture:path-suspect
AS_PATH: [65000 1199]
Failing hop: AS1199→AS65000 — AS65000 not in AS1199 ASPA provider set (only AS1103)

**What you will see in Grafana:**

- `path-suspect` counter jumps from 0 to 1
- The path-suspect routes table shows the leaked route with the failing hop

**CLI investigation:**

```bash
raven routes --posture path-suspect
```
PREFIX       PEER          ORIGIN  ROV    ASPA     POSTURE
145.102.136.0/22   10.0.0.1   AS1199  Valid  Invalid  path-suspect

```bash
raven routes --posture path-suspect --format json
```

The JSON output includes the full `aspa` field with `failing_hop` details
showing exactly which hop violated which ASPA record.

**Clean up:**

```bash
bash lab/demo-master.sh leak-clean
```

---

## Scenario 3 — Stealthy Hijack

**What it demonstrates:** A more-specific prefix announcement that diverts
traffic while the control plane appears clean. Detected only via data-plane
probing.

**Mechanism:** The attacker (AS65099) announces `203.0.113.0/25` — a
more-specific of the legitimate AS65000 /24. The edge router installs it via
LPM. RAVEN's BMP RIB for the /24 still shows AS65000 as origin — the control
plane looks clean. Only `raven check stealthy` reveals the divergence.

**Run:**

```bash
./demo-master.sh stealthy
```

**What RAVEN shows:**
Control plane (BMP/RIB):
  Route    : 203.0.113.0/24 via AS65000 (origin-only)
  RIB state: CLEAN — ROV Valid, no path violations
Data plane probe (traceroute from clab-raven-demo-edge):
  Probe 1  : TTL-exceeded at 10.0.3.2 (AS65099 — ATTACKER)
  Probe 2  : TTL-exceeded at 10.0.3.2 (AS65099 — ATTACKER)
  Probe 3  : TTL-exceeded at 10.0.3.2 (AS65099 — ATTACKER)
╔═════════════════════════════════════════════════════════╗
║ STEALTHY HIJACK DETECTED: control/data-plane divergence ║
║ RIB says: AS65000 │ Traffic goes to: AS65099            ║
╚═════════════════════════════════════════════════════════╝

**Why this matters:** This is the attack class documented in NDSS 2026
research and confirmed in the AS37100 incident (February 2025). BGP looking
glasses, route collectors, and RPKI dashboards all show a clean network.
Only data-plane probing correlated with BMP state reveals the compromise.

**Clean up:**

```bash
./demo-master.sh stealthy-clean
```

---

## Scenario 4 — RTR Cache Failure

**What it demonstrates:** RPKI validator unavailability and RAVEN's behavior
during cache staleness.

**Mechanism:** Routinator is killed. RAVEN detects the RTR session failure,
continues serving the last known RPKI state, and exposes staleness metrics
via Prometheus. Routinator is then restored and the session reconnects
automatically.

**Run:**

```bash
./demo-master.sh rtr-fail
```

**What RAVEN shows:**

After Routinator goes offline:
raven_rtr_session_state{cache="localhost:3323"} 0
raven_rtr_last_sync_seconds{cache="localhost:3323"} 1747600000
raven_rtr_vrp_count{cache="localhost:3323"} 870026

The route table continues to show correct validation results based on the
last known VRP and ASPA state. No routes are incorrectly accepted during
the outage.

**Why this matters:** Most routers default to fail-open on RTR expiry — they
stop enforcing ROV when the cache expires. RAVEN gives you advance warning:
monitor `raven_rtr_session_state` and alert when it hits 0.
`raven_rtr_last_sync_seconds` tells you exactly when the last successful
sync occurred.

---

## Scenario 5 — What-If Simulation

This scenario does not inject any attack. It shows the operator impact of
deploying routing security policies against the current route table.

```bash
bash lab/demo-master.sh whatif
```

This runs two simulations:

**reject-invalid simulation:**

```bash
raven what-if --reject-invalid
```

Shows how many routes would be dropped if you deployed reject-invalid policy
today. In the lab this will be a small number since the lab prefixes all have
ROAs. In production against a full DFZ table this is where the analysis becomes
valuable.

**ASPA enforcement simulation:**

```bash
raven what-if --aspa-enforce
```

Shows how many routes would be flagged as path-suspect under ASPA enforcement.
Also shows the large number of routes that are currently unverifiable due to
missing ASPA objects — which is the current state of the internet.

---

## Scenario 6 — ASPA Recommender

Analyse observed AS_PATHs and generate ASPA object suggestions:

```bash
bash lab/demo-master.sh recommend
```

This runs:

```bash
raven aspa recommend --min-observations 1
```

The recommender analyses every AS_PATH RAVEN has seen via BMP and infers
customer-provider relationships. It then cross-references these against the
ASPA store to identify which ASes could benefit from creating ASPA objects.

In the lab the output will reflect the small set of routes in the topology.
Against a production BMP feed with hundreds of thousands of routes, this
becomes a powerful tool for identifying gaps in ASPA coverage.

---

## Full Demo Narrative

For conference presentations, run the scenarios in this order:

1. `setup` — deploy the lab, show `raven status` — everything green
2. `baseline` — show the clean route table in `raven routes`
3. Open Grafana and `raven watch` side by side
4. `hijack` — inject the hijack, show real-time detection
5. `hijack-clean` — clean up, show the route disappearing
6. `stealthy` — announce a more-specific from the attacker, show `raven check stealthy`
   revealing the control/data-plane divergence
7. `stealthy-clean` — withdraw the more-specific
8. `leak` — inject the route leak, explain why ROV misses it but ASPA catches it
9. `leak-clean` — clean up
10. `rtr-fail` — kill Routinator, show staleness metrics and graceful behavior
11. `whatif` — show the policy impact simulation
12. `recommend` — show the ASPA recommender output
13. `down` — clean shutdown

---

## Phase 3 Scenarios

These scenarios demonstrate RAVEN's active response capabilities — the Event
Engine, webhooks, Flowspec lifecycle, and audit report.

### Scenario 7 — Webhook Alert

Shows the Event Engine firing a webhook when a hijack is detected.

**Setup** — start the webhook listener in a second terminal:

```bash
bash lab/demo-master.sh webhook-listen
```

**Inject the hijack:**

```bash
bash lab/demo-master.sh hijack
```

**What you will see in the webhook listener terminal:**
─── Alert: 2026-05-06 14:23:01 ───
{
"type": "new_route",
"prefix": "192.0.2.0/24",
"new_posture": "origin-invalid",
"rov_state": "Invalid",
"router_id": "10.0.0.1"
}

The webhook fires within milliseconds of BMP arrival — before any human
could have noticed the Grafana dashboard change.

**Clean up:**

```bash
bash lab/demo-master.sh hijack-clean
```

---

### Scenario 8 — Flowspec Lifecycle

Shows the full Flowspec mitigation lifecycle: automatic rule generation,
dry-run review, live GoBGP injection, and withdrawal.

```bash
bash lab/demo-master.sh flowspec
```

This runs interactively:

1. Injects the origin hijack to trigger a Flowspec rule
2. Shows the rule in `raven flowspec list` (dry-run mode)
3. Prompts you to toggle to live injection
4. Shows the rule in GoBGP after toggle
5. Prompts you to withdraw
6. Cleans up the hijack

**Key commands demonstrated:**

```bash
# Show active rules
raven flowspec list

# Toggle a rule from dry-run to live GoBGP injection
raven flowspec toggle "192.0.2.0/24|drop"

# Verify the rule is active in GoBGP
docker exec clab-raven-demo-gobgp gobgp global rib -a ipv4-flowspec

# Toggle back to dry-run (withdraw from GoBGP)
raven flowspec toggle "192.0.2.0/24|drop"
```

!!! warning
    The `flowspec` case requires the GoBGP sidecar container. Verify it is
    running with `docker ps | grep gobgp` before running this scenario.

---

### Scenario 9 — Audit Report

Shows `raven audit` generating a full security posture report for a router.

**Baseline audit:**

```bash
bash lab/demo-master.sh audit
```

This runs `raven audit --router 10.0.0.1` and shows:

- Total routes and per-posture breakdown
- ROV coverage percentage
- ASPA coverage percentage (0% in the lab — expected)
- Actionable recommendations

**Audit after hijack** — inject the hijack first, then audit:

```bash
bash lab/demo-master.sh hijack
raven audit --router 10.0.0.1
```

The audit now shows 2 origin-invalid routes and names the attacker ASN in
the top offenders section.

**Audit in different formats:**

```bash
# JSON output — pipe to your SIEM or ticket system
raven audit --router 10.0.0.1 --format json

# Markdown output — paste into incident post-mortems
raven audit --router 10.0.0.1 --format markdown
```

---

### Scenario 10 — Full Phase 3 Sequence

Runs all Phase 3 scenarios automatically in sequence:

```bash
bash lab/demo-master.sh phase3
```

This runs non-interactively:
1. Origin hijack detection + webhook
2. Flowspec rule generation + live toggle + GoBGP verification + withdraw
3. Audit report before and after hijack
4. Cleanup

Use this for rehearsing the full Phase 3 demo flow.
