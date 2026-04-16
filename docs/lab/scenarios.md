# Attack Scenarios

The demo lab includes three scripted scenarios that demonstrate RAVEN's
detection capabilities in real time. Run these after `demo-master.sh setup`
completes.

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

- The internet router (AS2121) announces `193.0.0.0/21` — a prefix it does
  not own and has no ROA for
- RAVEN receives this via BMP from the upstream router
- ROV lookup finds a ROA authorizing a different ASN
- RAVEN marks the route `origin-invalid`

**What you will see in `raven watch`:**
[14:23:01] NEW  193.0.0.0/21  via 172.20.20.2  AS2121  ROV:Invalid  posture:origin-invalid
Matched VRP: ROA{193.0.0.0/21, AS65000, maxLength=/21}
Reason: Origin AS2121 not authorized — ROA authorizes AS65000

**What you will see in Grafana:**

- `origin-invalid` counter jumps from 0 to 1
- The origin-invalid routes table shows the hijacked prefix

**CLI investigation:**

```bash
raven routes --posture origin-invalid
```
PREFIX          PEER          ORIGIN  ROV      ASPA     POSTURE
193.0.0.0/21    172.20.20.2   AS2121  Invalid  Unknown  origin-invalid

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
- The origin ASN is legitimate so ROV passes
- RAVEN walks the AS_PATH and finds a hop that violates an ASPA record
- RAVEN marks the route `path-suspect`

**What you will see in `raven watch`:**
[14:24:15] NEW  10.0.0.0/8  via 172.20.20.2  AS2121  ROV:Valid  ASPA:Invalid  posture:path-suspect
AS_PATH: [65000 2121 65000]
Failing hop: AS65000→AS2121 — AS2121 is not in ASPA(AS65000) provider set

**What you will see in Grafana:**

- `path-suspect` counter jumps from 0 to 1
- The path-suspect routes table shows the leaked route with the failing hop

**CLI investigation:**

```bash
raven routes --posture path-suspect
```
PREFIX       PEER          ORIGIN  ROV    ASPA     POSTURE
10.0.0.0/8   172.20.20.2   AS2121  Valid  Invalid  path-suspect

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

## Scenario 3 — What-If Simulation

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

## Scenario 4 — ASPA Recommender

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
6. `leak` — inject the route leak, explain why ROV misses it but ASPA catches it
7. `leak-clean` — clean up
8. `whatif` — show the policy impact simulation
9. `recommend` — show the ASPA recommender output
10. `down` — clean shutdown
