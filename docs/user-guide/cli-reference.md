# CLI Reference

RAVEN is CLI-first. Every command connects to a running RAVEN daemon via the
API (default: `localhost:11020`). Use `--address` to target a remote
instance; that instance must set
[`api.listen`](configuration.md#api-section) to an address you can reach,
because the API binds to loopback by default.

## Global Flags

These persistent flags are accepted by every command:

| Flag | Default | Description |
|---|---|---|
| `--address` | `localhost:11020` | RAVEN daemon address for CLI queries |
| `--config` | `./raven.yaml` | Config file path (search order: `./raven.yaml`, then `/etc/raven/raven.yaml`). Only `serve` reads the daemon config |
| `--log-level` | `info` | Log level: `debug`, `info`, `warn`, `error` |
| `--log-format` | `json` | Log format: `json` or `text` |

!!! note
    `--format` is **not** a global flag. It is a per-command flag: `table` or
    `json` on most commands, `wide` as well on `raven routes`, and `markdown`
    on `raven audit`. `raven status`, `raven peers` and `raven validate` have
    no `--format`.
    Each command's flag table below lists whether it accepts `--format`.

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

Show the route table summary, RPKI state and every BMP peer.

```bash
raven status
raven status --router edge1
```

**Flags:**

| Flag | Description |
|---|---|
| `--router` | List only this monitored router's peers: its BMP source address or its BMP sysName. The route table and RPKI totals stay daemon-wide |

`raven status` has no `--format` flag. For JSON, read the API directly on
the monitoring host:

```bash
curl -s http://127.0.0.1:11020/api/v1/status
```

**Example output:**

```text
RAVEN dev  uptime 2h14m5s

Route Table: 5 routes received (Adj-RIB-In pre-policy)
  path-suspect         2
  path-only            1
  secured              1
  unverified           1
By RIB:
  adj-rib-in-pre       5
  adj-rib-out-pre      41

RPKI: 1 VRPs (serial 4821)

BMP Peers: 4
  ROUTER     PEER                      ASN    STATE  ROUTES  OUT
  edge1      198.51.100.1              64500  up     2       41
  edge1      198.51.100.2              64496  up     1       0
  edge1      192.0.2.250               64510  up     1       0
  192.0.2.2  2001:db8::2 rd 65000:100  64505  up     1       0
```

**Route Table** and its posture breakdown count Adj-RIB-In pre-policy
routes only: what your routers received, before import policy, comparable
to the received-routes count on the router itself. Earlier releases added
every RIB together, so a router that also sent post-policy or Adj-RIB-Out
data inflated the total. The posture lines appear in no fixed
order. **By RIB** appears when the table holds more than one RIB type and
lists each one separately.

**BMP Peers** columns:

| Column | Meaning |
|---|---|
| `ROUTER` | The monitored router: its BMP sysName, or its BMP source address when it sent none |
| `PEER` | The BGP peer's address. A peer in a VRF shows the VRF's route distinguisher, as in `2001:db8::2 rd 65000:100` |
| `ASN` | The peer's ASN |
| `STATE` | `up` or `down` |
| `ROUTES` | The peer's Adj-RIB-In pre-policy routes now in the table |
| `OUT` | The peer's Adj-RIB-Out pre-policy routes: what your router sends that peer (RFC 8671). Shown only when some peer has Adj-RIB-Out routes |

The same peer address on two routers, or in two VRFs of one router, is two
rows, because RAVEN keys peers by router, peer distinguisher and address.
`ROUTES` and `OUT` stay in separate columns so that received and sent routes
are never added together.

---

## raven peers

List every BMP peer with session metadata.

```bash
raven peers
raven peers --router edge1
```

**Flags:**

| Flag | Description |
|---|---|
| `--router` | List only this monitored router's peers: its BMP source address or its BMP sysName |

`raven peers` has no `--format` flag; for JSON, read
`http://127.0.0.1:11020/api/v1/peers` on the monitoring host.

**Example output:**

```text
ROUTER     PEER                      ASN    ROUTER ID     STATE  ROUTES  OUT  UP SINCE
edge1      198.51.100.1              64500  198.51.100.1  up     2       41   2026-10-18T09:12:44Z
edge1      198.51.100.2              64496  198.51.100.2  up     1       0    2026-10-18T09:12:44Z
edge1      192.0.2.250               64510  192.0.2.250   up     1       0    2026-10-18T09:12:45Z
192.0.2.2  2001:db8::2 rd 65000:100  64505  203.0.113.5   up     1       0    2026-10-18T09:13:02Z
```

The `ROUTER`, `PEER`, `ROUTES` and `OUT` columns mean the same as in
[`raven status`](#raven-status). `ROUTER ID` is the peer's BGP identifier and
`UP SINCE` the time its session came up (RFC 3339, UTC).

### The MALFORMED column

When a peer has sent a malformed UPDATE that RFC 7606 answers with a
session reset, `raven peers` adds a `MALFORMED` column. It shows the reason
for that peer and `-` for every other peer:

```text
ROUTER  PEER          ASN    ROUTER ID     STATE  ROUTES  UP SINCE              MALFORMED
edge1   198.51.100.1  64500  198.51.100.1  up     2       2026-10-18T09:12:44Z  mp_reach_next_hop
edge1   198.51.100.2  64496  198.51.100.2  up     1       2026-10-18T09:12:44Z  -
```

A marked peer's routes in RAVEN may be incomplete: RAVEN withdrew what it
could parse from the bad UPDATE and expects the router to reset the BGP
session. The mark clears when the router reports the peer up again (BMP
Peer Up). The reason is a `raven_bmp_malformed_updates_total` reason, and
the API's `/api/v1/peers` output carries it as `malformed_input`,
`malformed_reason` and `malformed_since`. See
[Malformed BGP Input](../troubleshooting.md#malformed-bgp-input).

---

## raven routes

Query the route table. Without filters, RAVEN returns every Adj-RIB-In
pre-policy route.

```bash
raven routes
raven routes --prefix 203.0.113.0/24
raven routes --origin-asn 64502
raven routes --peer 198.51.100.1
raven routes --router edge1
raven routes --posture path-suspect
raven routes --prefix 203.0.113.0/24 --peer 198.51.100.1
raven routes --posture path-suspect --format wide
raven routes --rib adj-rib-out-pre --posture origin-invalid
raven routes --format json
```

**Flags:**

| Flag | Description |
|---|---|
| `--prefix` | Filter by prefix (exact match) |
| `--origin-asn` | Filter by origin ASN |
| `--peer` | Filter by BMP peer address. An IPv4 peer matches in plain or IPv4-mapped form |
| `--router` | Filter by monitored router: its BMP source address or its BMP sysName |
| `--posture` | Filter by security posture |
| `--rib` | The RIB to list: `adj-rib-in-pre` (default), `adj-rib-in-post`, `adj-rib-out-pre` or `adj-rib-out-post` |
| `--format` | Output format: `table` (default), `wide` or `json` |

Every filter you give applies, so `--prefix 203.0.113.0/24 --peer
198.51.100.1` returns that prefix from that one peer. Earlier releases
honored only the first filter present and silently ignored the rest.
`--router` with a sysName matches every router that reported that name.

`raven routes` lists one RIB at a time, Adj-RIB-In pre-policy unless you
pass `--rib`, and every other filter applies within it. Adj-RIB-In
pre-policy is the default because it is what your peers actually sent,
before your import policy changed or filtered it. Use `--rib
adj-rib-in-post` to see what survived your import policy, and `--rib
adj-rib-out-pre` to check what your router advertises, for example
`origin-invalid` routes your network is sending to its peers. A RIB shows
routes only if your routers export it over BMP.

The API takes the same filter as the `rib` query parameter on
`/api/v1/routes` and `/api/v1/watch`, with the same values. On
`/api/v1/routes` it defaults to `adj-rib-in-pre`; on `/api/v1/watch`, a
stream without `rib` includes every RIB. An unknown value returns HTTP 400.

**Posture values for `--posture`:**

`secured` `origin-only` `path-suspect` `path-only` `unverified` `origin-invalid`

**Example output:**

```text
PREFIX                    ROUTER     PEER                ORIGIN   ROV       ASPA     POSTURE
203.0.113.0/24            edge1      198.51.100.1        AS64502  Valid     Invalid  path-suspect
198.51.100.0/24           edge1      192.0.2.250         AS64503  NotFound  Valid    path-only
2001:db8:2000::/36        edge1      198.51.100.1        AS64503  NotFound  Invalid  path-suspect
203.0.113.0/24            edge1      198.51.100.2        AS64502  Valid     Valid    secured
2001:db8:3000::/36        192.0.2.2  2001:db8::2 path 2  AS64504  NotFound  Unknown  unverified
```

`ROUTER` is the router's sysName, or its address when it sent none. When the
peer uses ADD-PATH (RFC 7911), `PEER` adds the path ID, as in
`2001:db8::2 path 2`, because the peer can send several paths for one
prefix and RAVEN keeps each one.

### --format wide

`--format wide` adds three columns that explain each route's ASPA result:

```text
PREFIX                    ROUTER     PEER                ORIGIN   ROV       ASPA     POSTURE       ASPA PROCEDURE  PROCEDURE SOURCE  FAILING HOP
203.0.113.0/24            edge1      198.51.100.1        AS64502  Valid     Invalid  path-suspect  upstream        config-asn        64501->64500
198.51.100.0/24           edge1      192.0.2.250         AS64503  NotFound  Valid    path-only     upstream        config-address    -
2001:db8:2000::/36        edge1      198.51.100.1        AS64503  NotFound  Invalid  path-suspect  upstream        config-asn        -
203.0.113.0/24            edge1      198.51.100.2        AS64502  Valid     Valid    secured       downstream      config-address    -
2001:db8:3000::/36        192.0.2.2  2001:db8::2 path 2  AS64504  NotFound  Unknown  unverified    upstream        default           -
```

| Column | Meaning |
|---|---|
| `ASPA PROCEDURE` | `upstream` or `downstream`: the verification procedure RAVEN used for this route |
| `PROCEDURE SOURCE` | Where the procedure came from: `config-address` (an `aspa-peers` entry for the peer address), `config-asn` (an `aspa-peers` entry for the peer ASN) or `default` (`aspa-default-procedure`) |
| `FAILING HOP` | For an ASPA `Invalid`, the hop that failed, as `customer->provider`: the customer AS's ASPA record does not list that provider. `-` when no hop failed |

In the example, the first route is a leak: AS64501's ASPA record does not
list AS64500, so the hop `64501->64500` fails. The third route is `Invalid`
with no failing hop because it failed the neighbor-AS check; `--format json`
shows the reason in `aspa_reason`. The route from the transit provider
(`198.51.100.2`) is verified `downstream` because an `aspa-peers` entry maps
that address to `neighbor-role: provider`. See
[ASPA Path Verification](../getting-started/concepts.md#aspa-path-verification)
and [`validation.aspa-peers`](configuration.md#validationaspa-peers).

### --format json

`--format json` prints the API's response unchanged: an array of route
objects. This is the leaking route from the examples above:

```json
[
  {
    "prefix": "203.0.113.0/24",
    "router": "192.0.2.1",
    "router_name": "edge1",
    "peer": "198.51.100.1",
    "peer_asn": 64500,
    "rib": "adj-rib-in-pre",
    "origin_asn": 64502,
    "as_path": [
      64500,
      64501,
      64502
    ],
    "next_hop": "198.51.100.1",
    "rov": "Valid",
    "rov_reason": "matches VRP {203.0.113.0/24, AS64502, /24}",
    "aspa": "Invalid",
    "aspa_procedure": "upstream",
    "aspa_procedure_source": "config-asn",
    "aspa_failing_hop": {
      "customer_asn": 64501,
      "provider_asn": 64500
    },
    "posture": "path-suspect",
    "timestamp": "2026-10-18T14:02:31Z"
  }
]
```

A route from a VRF peer with ADD-PATH carries `peer_distinguisher` and
`path_id`; a route that failed the neighbor-AS check carries `aspa_reason`:

```json
{
  "prefix": "2001:db8:3000::/36",
  "router": "192.0.2.2",
  "peer_distinguisher": "65000:100",
  "peer": "2001:db8::2",
  "peer_asn": 64505,
  "path_id": 2,
  "rib": "adj-rib-in-pre",
  ...
}
```

```json
{
  "prefix": "2001:db8:2000::/36",
  ...
  "aspa": "Invalid",
  "aspa_procedure": "upstream",
  "aspa_procedure_source": "config-asn",
  "aspa_reason": "neighbor-AS check failed: first AS in AS_PATH is AS64511, peer is AS64500",
  ...
}
```

**Route fields:**

| Field | Description |
|---|---|
| `prefix` | The route's prefix |
| `router` | The monitored router: the source address of its BMP session |
| `router_name` | The router's BMP Initiation sysName. Omitted when the router sent none |
| `peer_distinguisher` | The peer's VRF route distinguisher, from the BMP per-peer header. Omitted for peers in the global instance |
| `peer` | The BGP peer's address |
| `peer_asn` | The BGP peer's ASN |
| `path_id` | The ADD-PATH path identifier (RFC 7911). Omitted when ADD-PATH is not in effect for that peer and address family |
| `rib` | The RIB the route is in: `adj-rib-in-pre`, `adj-rib-in-post`, `adj-rib-out-pre` or `adj-rib-out-post`. Every route in one response has the RIB you asked for with `--rib` |
| `origin_asn` | The origin AS: the last AS of the AS_PATH's final AS_SEQUENCE (RFC 6811 §2). `0` when the route has no origin: an empty AS_PATH, a path ending in an AS_SET (origin NONE), or a path ending in a confederation segment. The table shows `-` |
| `as_path` | The AS_PATH, neighbor first, origin last |
| `next_hop` | The BGP next hop |
| `rov` | `Valid`, `Invalid` or `NotFound` |
| `rov_reason` | Why ROV reached its result |
| `aspa` | `Valid`, `Invalid`, `Unknown` or `Unverifiable` |
| `aspa_procedure` | `upstream` or `downstream` |
| `aspa_procedure_source` | `config-address`, `config-asn` or `default` |
| `aspa_reason` | Why ASPA reached its result when no hop explains it: the neighbor-AS check failed, the route is an Adj-RIB-Out route, or its AS_PATH has confederation segments. RAVEN does not ASPA-verify the last two. Omitted otherwise |
| `aspa_failing_hop` | `{customer_asn, provider_asn}`: the hop that made ASPA `Invalid`. Omitted when no hop failed |
| `posture` | The combined [security posture](security-postures.md) |
| `timestamp` | When the monitored router received the route, from the BMP per-peer header, in RFC 3339 UTC. If the router sends a zero timestamp, RAVEN uses the time it received the BMP message instead |

`timestamp` comes from the router, not from RAVEN's clock: it is when the
monitored router received the route (RFC 7854 §4.2), not when RAVEN read
it. When RAVEN restarts or a BMP session reconnects, the router resends its
whole table, and a timestamp taken by RAVEN would make every route look
seconds old. The router's timestamp keeps the route's age, so for an
`origin-invalid` route it tells you how long the hijacked prefix has been in
your router's table, which is what you need to judge impact. The router
decides what it puts in this field, and a router that sends 0 gets RAVEN's
receive time instead; compare a few routes against the router's own
"received" time before you rely on it for route age.

---

## raven validate

Show how RAVEN validated the routes it holds for one prefix, with the reason
for each ROV result. Use it to answer "why is this prefix `origin-invalid`?"
from the daemon's live state.

```bash
raven validate --prefix 203.0.113.0/24
```

`raven validate` reads the route table of a running daemon, so it needs
`raven serve` running and the prefix received over BMP. It does not validate
a prefix the routers have not sent; for a prefix RAVEN has not seen, it
prints `no routes found for prefix 203.0.113.0/24`.

**Flags:**

| Flag | Description |
|---|---|
| `--prefix` | Prefix to look up, exact match (required) |
| `--origin-asn` | Not supported yet: `raven validate` exits with an error if you pass it |
| `--rtr` | Not supported yet: `raven validate` exits with an error if you pass it |

Both flags are reserved for one-shot validation against an RTR cache, which
does not exist yet. Earlier releases accepted them and silently ignored
them; RAVEN now refuses them so that you never read a result for an origin
you did not get. With either flag, `raven validate` prints, and exits with
status 1:

```text
Error: --origin-asn is not supported yet; validate reads the running daemon's routes for --prefix
```

```text
Error: --rtr is not supported yet; validate reads the running daemon's routes for --prefix
```

`raven validate` has no `--format` flag. For JSON, use
`raven routes --prefix 203.0.113.0/24 --format json`.

**Example output:**

```text
PREFIX                    PEER          ORIGIN   ROV      POSTURE         REASON
203.0.113.0/24            198.51.100.1  AS64502  Valid    secured         matches VRP {203.0.113.0/24, AS64502, /24}
203.0.113.0/24            198.51.100.9  AS64511  Invalid  origin-invalid  origin AS64511 not authorized by any covering VRP
```

There is one row per route RAVEN holds for the prefix: one per peer, per
router and per ADD-PATH path. `REASON` is the ROV reason: the VRP that
matched, the origin AS that no covering VRP authorizes, or
`no covering VRPs found`. Like `raven routes`, it lists Adj-RIB-In
pre-policy routes.

---

## raven watch

Stream route updates to your terminal as RAVEN validates them. Runs until you
press Ctrl+C.

```bash
raven watch
raven watch --posture path-suspect
raven watch --router edge1 --posture origin-invalid
raven watch --peer 198.51.100.1
raven watch --prefix 203.0.113.0/24 --origin-asn 64502
```

**Flags:**

| Flag | Description |
|---|---|
| `--prefix` | Only stream routes for this prefix |
| `--origin-asn` | Only stream routes with this origin ASN |
| `--peer` | Only stream routes from this BMP peer |
| `--router` | Only stream routes from this monitored router: its address or BMP sysName |
| `--posture` | Only stream routes with this posture |

Every filter you give applies, as with `raven routes`. Earlier releases sent
only `--posture` to the daemon and ignored the other filters.

**Example output:**

```text
watching for route events... (Ctrl+C to stop)

[path-suspect] 203.0.113.0/24 via 198.51.100.1 origin:AS64502 AS_PATH:[64500 64501 64502] rov:Valid posture:path-suspect
```

!!! tip
    Run `raven watch --posture path-suspect` in a spare terminal during an
    incident. It shows leaked routes the moment RAVEN validates them.

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

## raven check stealthy

Detects stealthy BGP hijacks by comparing the control-plane BMP RIB view
against actual data-plane forwarding using probes. A diverging first-hop ASN
indicates traffic is being routed through a different network than BGP
advertises — invisible to the control plane alone.

```bash
raven check stealthy --prefix 203.0.113.0/24
raven check stealthy --prefix 203.0.113.0/24 --probe-via clab-raven-demo-edge
raven check stealthy --prefix 203.0.113.0/24 --probe-count 5 --timeout 5s
raven check stealthy --prefix 203.0.113.0/24 --format json
```

**Flags:**

| Flag | Default | Description |
|---|---|---|
| `--prefix` | required | Prefix to probe (CIDR) |
| `--probe-count` | 3 | Number of probes to send |
| `--timeout` | 3s | Probe timeout per probe |
| `--probe-via` | — | Run traceroute from inside a Docker container (for topologies where the host is outside BGP forwarding) |
| `--format` | table | Output format: `table` or `json` |

**Verdicts:**

| Verdict | Meaning |
|---|---|
| `CLEAN` | Data-plane first hop matches BMP RIB expected AS |
| `STEALTHY HIJACK DETECTED` | First hop AS differs from BMP RIB — traffic diverted |
| `DIVERGENCE` | First hop not in BMP peer table — unconfirmed divergence |
| `INCONCLUSIVE` | All probes timed out |

**Example — clean baseline:**
$ raven check stealthy --prefix 203.0.113.0/24 --probe-via clab-raven-demo-edge
Checking control/data-plane divergence for 203.0.113.0/24...
Control plane (BMP/RIB):
  Route    : 203.0.113.0/24 via AS65000 (origin-only)
  RIB state: CLEAN — ROV Valid, no path violations
Data plane probe (traceroute from clab-raven-demo-edge to 203.0.113.1):
  Probe 1  : TTL-exceeded at 10.0.0.1 (AS65000 — expected)
  Probe 2  : TTL-exceeded at 10.0.0.1 (AS65000 — expected)
  Probe 3  : TTL-exceeded at 10.0.0.1 (AS65000 — expected)
╔══════════════════════════════════════════════════════╗
║ CLEAN: Data-plane path matches BMP RIB               ║
║ RIB says: AS65000 │ Traffic goes to: AS65000         ║
╚══════════════════════════════════════════════════════╝

**Example — stealthy hijack detected:**
$ raven check stealthy --prefix 203.0.113.0/24 --probe-via clab-raven-demo-edge
Checking control/data-plane divergence for 203.0.113.0/24...
Control plane (BMP/RIB):
  Route    : 203.0.113.0/24 via AS65000 (origin-only)
  RIB state: CLEAN — ROV Valid, no path violations
Data plane probe (traceroute from clab-raven-demo-edge to 203.0.113.1):
  Probe 1  : TTL-exceeded at 10.0.3.2 (AS65099 — ATTACKER)
  Probe 2  : TTL-exceeded at 10.0.3.2 (AS65099 — ATTACKER)
  Probe 3  : TTL-exceeded at 10.0.3.2 (AS65099 — ATTACKER)
╔═════════════════════════════════════════════════════════╗
║ STEALTHY HIJACK DETECTED: control/data-plane divergence ║
║ RIB says: AS65000 │ Traffic goes to: AS65099            ║
╚═════════════════════════════════════════════════════════╝

**Note on permissions:** Raw ICMP requires `CAP_NET_RAW` or root. Grant it
without sudo:

```bash
sudo setcap cap_net_raw=eip ./raven
```

If ICMP fails, RAVEN automatically falls back to TCP SYN probes on port 80.
Use `--probe-via` to run probes from inside a container that is part of the
BGP topology.

---

## raven check global

Correlates a prefix's local BMP-observed origin against the origin ASNs that
third-party route collectors see for it, via the
[RIPEstat looking-glass data call](https://stat.ripe.net/docs/02.data-api/looking-glass.html).

RAVEN's ROV and ASPA validation is local-vantage-point only: it knows what
your own BMP-attached routers received. That cannot tell you whether a
suspicious route is a real hijack the rest of the internet also sees, or a
purely local leak or misconfiguration. This check answers that question.

```bash
raven check global --prefix 203.0.113.0/24
raven check global --prefix 203.0.113.0/24 --origin-asn 64511
raven check global --prefix 203.0.113.0/24 --format json
```

**Flags:**

| Flag | Default | Description |
|---|---|---|
| `--prefix` | required | Prefix to correlate (CIDR) |
| `--origin-asn` | — | Origin ASN to compare against. Defaults to the daemon's BMP-observed origin for the prefix |
| `--format` | table | Output format: `table` or `json` |

**Verdicts:**

| Verdict | Meaning |
|---|---|
| `MATCH` | The local origin is the single most-observed origin globally — the route looks the same from outside |
| `DIVERGENT` | The world's majority origin is not the local origin, or the local origin is only tied for most-observed (a MOAS conflict) |
| `LOCAL_ONLY` | No collector sees this prefix at all — points at a local leak or misconfiguration, not a propagated hijack |
| `INCONCLUSIVE` | The query failed, timed out, returned malformed data, or there was no local origin to compare against |

`MATCH` requires a strict majority. A tie between the local origin and
another ASN is reported as `DIVERGENT`: a MOAS conflict at equal global
visibility still warrants a look.

**Example — global match:**

```
$ raven check global --prefix 8.8.8.0/24
Correlating global visibility for 8.8.8.0/24...

Local view (BMP/RIB):
  Route   : 8.8.8.0/24 via AS15169 (secured)
  Peer    : 10.0.0.1 (AS65000), 1 route(s) in RIB
  Local   : ROV Valid, ASPA Valid

Global view (RIPEstat looking-glass, https://stat.ripe.net):
  ORIGIN   COLLECTOR PEERS  SHARE  NOTE
  AS15169  368              100%   matches local
  368 collector peer observations in 608ms

╔═══════════════════════════════════════════════════════════════════════════════════╗
║ GLOBAL MATCH: Global majority origin matches local BMP view (368 collector peers) ║
║ Local RIB says: AS15169 │ The internet says: AS15169                              ║
╚═══════════════════════════════════════════════════════════════════════════════════╝
```

**Example — global divergence (the hijack is real and propagated):**

```
$ raven check global --prefix 203.0.113.0/24
Correlating global visibility for 203.0.113.0/24...

Local view (BMP/RIB):
  Route   : 203.0.113.0/24 via AS64511 (origin-invalid)
  Peer    : 10.0.3.2 (AS65099), 1 route(s) in RIB
  Local   : ROV Invalid, ASPA Unknown

Global view (RIPEstat looking-glass, https://stat.ripe.net):
  ORIGIN   COLLECTOR PEERS  SHARE  NOTE
  AS65000  18               82%
  AS64511  4                18%    matches local
  22 collector peer observations in 512ms

╔════════════════════════════════════════════════════════════════════════════════════════╗
║ GLOBAL DIVERGENCE DETECTED: MOAS conflict — 2 origins seen globally, most-observed ...  ║
║ Local RIB says: AS64511 │ The internet says: AS65000                                    ║
╚════════════════════════════════════════════════════════════════════════════════════════╝
```

**Example — local-only route (a leak, not a hijack):**

```
$ raven check global --prefix 203.0.113.0/24 --origin-asn 64511
Global view (RIPEstat looking-glass, https://stat.ripe.net):
  no observations — no route collector carries this prefix

╔═════════════════════════════════════════════════════════════════════════════════════════════════════════════╗
║ LOCAL-ONLY ROUTE: No collector sees this prefix — a local leak or misconfiguration, not a propagated hijack  ║
║ Local RIB says: AS64511 │ The internet says: nobody                                                         ║
╚═════════════════════════════════════════════════════════════════════════════════════════════════════════════╝
```

**Notes:**

- The looking-glass call is **exact-match**. A more-specific prefix that no
  collector carries returns no observations, which is exactly the
  `LOCAL_ONLY` signal — not a query failure.
- **Collector peers, not collectors.** Counts are in units of one
  (route collector, peer address) pair, so a single RIS collector peering
  with 30 networks contributes up to 30 observations.
- This command reads `external.ripestat` from `raven.yaml` for the base URL,
  timeout, cache TTL and rate limit, but does **not** require
  `external.ripestat.enabled: true` — invoking it is itself the explicit
  opt-in. The `enabled` flag gates only the Event Engine's automatic
  correlation. See
  [Configuration → External Correlation](configuration.md#external-correlation-ripestat).
- Without `--origin-asn` the baseline comes from the daemon's RIB, so
  `raven serve` must be reachable. With `--origin-asn` the check works
  against no daemon at all.
- It is **fail-open**: an unreachable or malformed RIPEstat degrades to
  `INCONCLUSIVE` and exits 0. The result is a standalone annotation and does
  not change the route's [security posture](security-postures.md).

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

---

## raven rtr

Parent command group for RTR (RPKI-to-Router) cache session tools. It has no
action of its own — run it bare to list the subcommands.

```bash
raven rtr
```

**Subcommands:**

| Command | Description |
|---|---|
| `raven rtr monitor` | Observe an RTR cache session, log telemetry to NDJSON, and detect anomalies |
| `raven rtr seed-baseline` | Build an anomaly-detector baseline snapshot from recorded telemetry |

These commands are **standalone** — they connect directly to an RTR cache and
do not require a running `raven serve` daemon or a config file.

---

## raven rtr monitor

Connect to a single RTR cache, record session lifecycle and sync events as
NDJSON, and run the adaptive anomaly detector over the sync stream. It keeps
in-memory VRP/ASPA stores only to drive the RTR client — no BMP ingestion or
route validation happens here. Every setting comes from flags; no config file
is read.

```bash
raven rtr monitor --cache localhost:3323 --log-file rtr-telemetry.ndjson
raven rtr monitor --cache rpki.example.com:3323 --transport tls
raven rtr monitor --log-file /tmp/rtr.ndjson --prometheus :9595
raven rtr monitor --anomaly-snapshot ~/.raven/anomaly-baseline.json --prometheus :9595
```

Runs in the foreground until you press Ctrl+C.

**Running instance:** [ASPA Explorer](https://aspa.ping6.it/statistics.html#rtrtitle), a public service of the Department of Information Engineering at the University of Padova, runs `raven rtr monitor` against its own Routinator and Fort validators and several public RTR servers, and publishes a summary of the telemetry every five minutes (also available as [JSON](https://aspa.ping6.it/api/v1/rtr.json)).

**Flags:**

| Flag | Default | Description |
|---|---|---|
| `--cache` | `localhost:3323` | RTR cache address |
| `--log-file` | `rtr-telemetry.ndjson` | Path to the NDJSON telemetry output file (created if absent, appended if present) |
| `--transport` | `tcp` | RTR transport: `tcp` or `tls` |
| `--prometheus` | — | Prometheus listen address (e.g. `:9595`); empty disables the metrics endpoint |
| `--anomaly-snapshot` | `~/.raven/anomaly-baseline.json` | Path to the adaptive anomaly-detector baseline snapshot (JSON) |

**Anomaly detection:**

- The detector **warm-starts** from the baseline snapshot at `--anomaly-snapshot`
  if one exists. A missing or unreadable snapshot degrades to a cold start —
  it never blocks startup. Seed a snapshot ahead of time with
  `raven rtr seed-baseline` to skip the live warm-up.
- Each incremental sync is scored against rolling per-metric windows
  (`interval_since_ns`, `sync_duration_ns`, `vrp_announced`, `vrp_withdrawn`,
  `aspa_announced`, `aspa_withdrawn`) using median/MAD adaptive thresholds.
- **Full resyncs are recorded but not evaluated** — a full table dump reports
  nearly the entire VRP/ASPA set as "announced", which is not comparable to
  incremental deltas and would trip a false positive.
- When an anomaly fires it is written to the NDJSON sink as an `anomaly` record
  and, if `--prometheus` is set, increments `raven_rtr_anomaly_total{cache,severity}`
  and sets `raven_rtr_anomaly_last_timestamp{cache}`. See
  [Observability & Metrics](observability.md) for the metric reference.
- The baseline is saved periodically and again on graceful shutdown, so a
  restarted monitor resumes from where it left off.

**Watching for anomalies live:**

```bash
tail -f rtr-telemetry.ndjson | grep '"event_type":"anomaly"'
```

---

## raven rtr seed-baseline

Replay a recorded NDJSON telemetry file through the anomaly detector's rolling
windows and write the resulting baseline snapshot. Run it once against an
existing collection to produce a reusable baseline that
`raven rtr monitor --anomaly-snapshot` loads on startup, skipping the live
warm-up period.

```bash
raven rtr seed-baseline --input rtr-baseline.ndjson
raven rtr seed-baseline --input rtr-baseline.ndjson --output ~/.raven/anomaly-baseline.json
```

Only **sync** records populate the windows; connected/disconnected and
structural (reset/error) records are ignored, and full syncs are skipped. The
input is treated as ground-truth-normal, so no anomalies are reported during
seeding.

**Flags:**

| Flag | Default | Description |
|---|---|---|
| `--input` | required | Path to the recorded NDJSON telemetry file to seed from |
| `--output` | `~/.raven/anomaly-baseline.json` | Path to write the baseline snapshot to |

**Example output:**
lines processed: 40320
sync records:    1342 (7 full syncs skipped)
window fill levels for localhost:3323:
  interval_since_ns: 500/500 (full)
  sync_duration_ns:  500/500 (full)
  vrp_announced:     500/500 (full)
  vrp_withdrawn:     500/500 (full)
  aspa_announced:    500/500 (full)
  aspa_withdrawn:    500/500 (full)
output: /home/user/.raven/anomaly-baseline.json

A window that reaches its cap (500 samples) reports `(full)` — the detector has
a complete baseline for that metric. Partially-filled windows still work but
give the detector less history to reason about.

!!! tip
    The typical workflow is: record telemetry with `raven rtr monitor` over a
    representative period, run `raven rtr seed-baseline` against that file once,
    then start the live monitor with `--anomaly-snapshot` pointed at the seeded
    baseline so it detects from the first sync.
