# CLI Reference

RAVEN is CLI-first. Every command connects to a running RAVEN daemon via the
API (default: `localhost:11020`). Use `--address` to target a remote instance.

## Global Flags

| Flag | Default | Description |
|---|---|---|
| `--address` | `localhost:11020` | RAVEN daemon address |
| `--config` | `raven.yaml` | Config file path (serve command only) |
| `--format` | `table` | Output format: `table` or `json` |

---

## raven serve

Start the RAVEN daemon — BMP listener, RTR client, validation engine, and all
configured outputs.

```bash
raven serve
raven serve --config /etc/raven/raven.yaml
raven serve --config raven.yaml
```

RAVEN runs in the foreground. Use systemd, screen, or tmux for production.

**What happens on startup:**

1. Loads config file
2. Starts Prometheus metrics endpoint
3. Starts API server
4. Connects to RTR caches — downloads VRPs and ASPA data
5. Starts BMP listener — waits for router connections
6. As routers connect, receives table dumps and begins validation

---

## raven status

Show the health of all BMP peers and RTR caches.

```bash
raven status
raven status --format json
```

**Example output:**
BMP PEERS
PEER           ASN     STATE  ROUTES   SINCE
192.168.1.1    65001   UP     842341   5m ago
192.168.1.2    65002   UP     839102   5m ago
RTR CACHES
CACHE                STATE  SERIAL  VRPS    ASPAS  LAST SYNC
localhost:3323        UP     4821    542000  1407   23s ago

---

## raven peers

List all BMP peers with session metadata.

```bash
raven peers
raven peers --format json
```

**Example output:**
PEER          ASN     ROUTER-ID    STATE  ROUTES   AFI        SINCE
192.168.1.1   65001   10.0.0.1     UP     842341   IPv4+IPv6  5m ago
192.168.1.2   65002   10.0.0.2     UP     839102   IPv4       5m ago

---

## raven routes

Query the route table. Without filters, returns all routes. Use filters to
narrow results.

```bash
raven routes
raven routes --prefix 1.0.0.0/24
raven routes --origin-asn 13335
raven routes --peer 192.168.1.1
raven routes --posture origin-invalid
raven routes --posture path-suspect
raven routes --format json
```

**Flags:**

| Flag | Description |
|---|---|
| `--prefix` | Filter by prefix (exact match) |
| `--origin-asn` | Filter by origin ASN |
| `--peer` | Filter by BMP peer IP address |
| `--posture` | Filter by security posture |
| `--afi` | Filter by address family: `ipv4` or `ipv6` |
| `--format` | Output format: `table` or `json` |

**Posture values for `--posture`:**

`secured` `origin-only` `path-suspect` `path-only` `unverified` `origin-invalid`

**Example output:**
PREFIX           PEER          ORIGIN   ROV      ASPA     POSTURE
1.0.0.0/24       192.168.1.1   AS13335  Valid    Valid    secured
8.8.8.0/24       192.168.1.1   AS15169  Valid    Unknown  origin-only
192.0.2.0/24     192.168.1.2   AS64666  Invalid  Unknown  origin-invalid

---

## raven validate

One-shot validation of a prefix and origin ASN against current RPKI state.
Does not require a BMP session — useful for quick checks.

```bash
raven validate --prefix 1.0.0.0/24 --origin-asn 13335
raven validate --prefix 192.0.2.0/24 --origin-asn 64666
raven validate --prefix 1.0.0.0/24 --origin-asn 13335 --format json
```

**Flags:**

| Flag | Description |
|---|---|
| `--prefix` | Prefix to validate (required) |
| `--origin-asn` | Origin ASN to validate against (required) |
| `--format` | Output format: `table` or `json` |

**Example output:**
PREFIX         ORIGIN    ROV     MATCHED VRP
1.0.0.0/24     AS13335   Valid   ROA{1.0.0.0/24, AS13335, maxLength=/24}

---

## raven watch

Stream live validation state changes to your terminal. Runs continuously until
you press Ctrl+C.

```bash
raven watch
raven watch --posture origin-invalid
raven watch --posture path-suspect
raven watch --peer 192.168.1.1
raven watch --prefix 1.0.0.0/24
```

**Flags:**

| Flag | Description |
|---|---|
| `--posture` | Only stream routes matching this posture |
| `--peer` | Only stream routes from this peer |
| `--prefix` | Only stream routes for this prefix |
| `--format` | Output format: `table` or `json` |

**Example output:**
[14:23:01] NEW    192.0.2.0/24  via 192.168.1.2  AS64666  ROV:Invalid  ASPA:Unknown  posture:origin-invalid
[14:23:04] CHANGE 10.0.0.0/8    via 192.168.1.1  AS64501  ROV:Valid    ASPA:Invalid  posture:path-suspect
AS_PATH: [64501 3356 7018]  failing hop: AS3356→AS7018 not in ASPA(AS3356)

!!! tip
    Run `raven watch --posture origin-invalid` in a terminal during a demo or
    incident. It will catch hijacks and ROV state changes the moment they happen.

---

## raven aspa

Show ASPA records for a specific ASN from the local ASPA store.

```bash
raven aspa --asn 13335
raven aspa --asn 13335 --format json
```

**Flags:**

| Flag | Description |
|---|---|
| `--asn` | ASN to look up (required) |
| `--format` | Output format: `table` or `json` |

**Example output:**
ASPA RECORD — AS13335
─────────────────────────────────
Provider set (3):
AS174
AS1299
AS3356

If the ASN has no ASPA record in the current RTR cache, RAVEN will say so.

---

## raven aspa recommend

Analyse observed AS_PATHs from your BMP feeds and suggest which ASPA objects
your neighbours should create. Based on heuristics — verify with your peers
before acting on suggestions.

```bash
raven aspa recommend
raven aspa recommend --peer 192.168.1.1
raven aspa recommend --asn 64500
raven aspa recommend --min-observations 10
raven aspa recommend --top 20
raven aspa recommend --format json
```

**Flags:**

| Flag | Description |
|---|---|
| `--peer` | Restrict analysis to routes from a specific peer IP |
| `--asn` | Show suggestions for a specific customer ASN |
| `--min-observations` | Minimum route observations to include a relationship (default: 1) |
| `--top` | Show top N suggestions, default 20 (0 = all) |
| `--include-existing` | Include ASNs that already have complete ASPA coverage |
| `--format` | Output format: `table` or `json` |

**Example output:**
ASPA OBJECT RECOMMENDATIONS (3)
Note: heuristic — based on observed AS_PATHs only. Verify with your peers.
┌─ AS64500  [no ASPA record]  confidence: 80%  observations: 1423
│  SUGGESTED PROVIDER    OBSERVED      STATUS
│  AS1299                892 routes    ✗ MISSING
│  AS3356                531 routes    ✗ MISSING
└──────────────────────────────────────────────────
┌─ AS64501  [existing ASPA (1 providers)]  confidence: 70%  observations: 203
│  SUGGESTED PROVIDER    OBSERVED      STATUS
│  AS174                 203 routes    ✗ MISSING
│  AS1299                203 routes    ✓ in ASPA
└──────────────────────────────────────────────────

**Confidence score** is a heuristic based on observation count and provider
count. Higher is more reliable. Scores above 70% are generally trustworthy.
Always verify with your peers before creating ASPA objects.

---

## raven what-if

Simulate the impact of deploying routing security policies without changing
anything on your routers.

### --reject-invalid

Simulate deploying reject-invalid ROV policy:

```bash
raven what-if --reject-invalid
raven what-if --reject-invalid --peer 192.168.1.1
raven what-if --reject-invalid --format json
```

**Example output:**
REJECT-INVALID IMPACT SUMMARY
─────────────────────────────────────────────────
Routes currently received:     1,044,293
Routes that would be rejected: 2,341 (0.22%)
Prefixes affected:             1,892
Unique origin ASNs affected:   487
TOP AFFECTED ORIGIN ASNs
ORIGIN ASN   ROUTES  PREFIXES  REASON
AS64666      142     142       Origin AS mismatch
AS64667      89      89        maxLength exceeded
AS64668      71      71        Origin AS mismatch

### --aspa-enforce

Simulate dropping all path-suspect routes:

```bash
raven what-if --aspa-enforce
raven what-if --aspa-enforce --format json
```

**Example output:**
ASPA ENFORCEMENT IMPACT SUMMARY
─────────────────────────────────────────────────
Routes currently received:     1,044,293
Routes with path-suspect:      12 (0.00%) ← would be dropped
Routes with no ASPA coverage:  891,203 (85.3%) ← unverifiable today
Prefixes affected:             10
Unique origin ASNs affected:   8
TOP FAILING AS_PATH HOPS
CUSTOMER ASN    UNAUTH PROVIDER  ROUTES AFFECTED
AS3356          AS7018           8
AS1299          AS3257           4

**Flags:**

| Flag | Description |
|---|---|
| `--reject-invalid` | Simulate reject-invalid ROV policy |
| `--aspa-enforce` | Simulate ASPA enforcement policy |
| `--peer` | Restrict simulation to routes from a specific peer |
| `--format` | Output format: `table` or `json` |

---

## raven version

Show RAVEN version and build information.

```bash
raven version
```
raven version v0.1.0 (commit: abc1234, built: 2026-04-15T00:00:00Z)

---

## raven audit

Generate a read-only security posture report for a router. Shows ROV/ASPA
coverage, per-peer posture breakdown, top offending ASNs, and actionable
recommendations.

```bash
raven audit --router 10.0.0.1
raven audit --router 10.0.0.1 --format json
raven audit --router 10.0.0.1 --format markdown
```

**Flags:**

| Flag | Description |
|---|---|
| `--router` | Peer IP address to audit (required) |
| `--format` | Output format: `table`, `json`, or `markdown` |

**Example output:**
Router: 10.0.0.1
Total Routes: 5
ROV Coverage:  80.0%
ASPA Coverage: 0.0%
POSTURE          COUNT
origin-only      3   (60%)
unverified       1   (20%)
origin-invalid   1   (20%)
TOP OFFENDERS
AS65001   1 route   origin-invalid   10.10.0.0/24
RECOMMENDATIONS

1 origin-invalid route — run raven what-if --reject-invalid
ASPA coverage 0% — register ASPA objects at your RIR


!!! tip
    Run `raven audit` weekly in a cron job with `--format json` to track
    your security posture over time. Use `--format markdown` for NOC reports
    and incident post-mortems.

---

## raven flowspec list

Show all active Flowspec mitigation rules in RAVEN's registry. Displays
the rule key, action, dry-run state, remaining TTL, and insertion time.

```bash
raven flowspec list
raven flowspec list --format json
```

**Flags:**

| Flag | Description |
|---|---|
| `--format` | Output format: `table` or `json` |

**Example output:**
RAVEN Flowspec Rules
════════════════════════════════════════════════════════
KEY                            ACTION  DRY-RUN  TTL        INSERTED
192.0.2.0/24|drop              drop    yes      4m50s      2026-05-06 14:23:01
10.10.0.0/24|drop              drop    yes      3m55s      2026-05-06 14:24:23
2 active rules  (dry-run mode — no live injection)

Rules are created automatically by the Event Engine when a route matches
a configured `flowspec` action. Rules in `dry-run` mode are tracked but
not injected into GoBGP until toggled live with `raven flowspec toggle`.

---

## raven flowspec toggle

Toggle a Flowspec rule between dry-run and live GoBGP injection. When
toggled to live, RAVEN calls GoBGP's AddPath API to inject the rule.
Toggle again to withdraw and return to dry-run.

```bash
raven flowspec toggle "192.0.2.0/24|drop"
```

**Arguments:**

| Argument | Description |
|---|---|
| `<key>` | Rule key as shown in `raven flowspec list` (required) |

**Example — toggling to live:**
Rule: 192.0.2.0/24|drop
Dry-run: yes → LIVE
⚠  Live injection enabled. Flowspec rule is now active in GoBGP.
Run 'raven flowspec toggle "192.0.2.0/24|drop"' again to
withdraw and return to dry-run mode.

**Example — withdrawing:**
Rule: 192.0.2.0/24|drop
Dry-run: LIVE → yes (withdrawn from GoBGP)

!!! warning
    Live injection sends a Flowspec route to GoBGP which will propagate
    it to any configured BGP peers. Only toggle to live when you intend
    to apply the mitigation. The rule will auto-expire after the
    configured TTL (default: 5 minutes).
