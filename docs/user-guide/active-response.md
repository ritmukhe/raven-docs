# Active Response

RAVEN's Phase 3 capabilities move beyond observation into automated response.
The Event Engine evaluates configurable rules on every posture change and
triggers actions — webhooks, logs, and Flowspec mitigation rules.

---

## Event Engine

The Event Engine runs continuously alongside the validation pipeline. On every
posture change it evaluates all configured rules and fires matching actions.

### Rule Structure

```yaml
events:
  rules:
    - name: "alert-origin-invalid"
      trigger:
        type: posture_change
        postures: ["origin-invalid"]
      cooldown: 60s
      actions:
        - type: log
          level: warn
        - type: webhook
          url: "https://hooks.example.com/raven"
        - type: flowspec
          dry_run: true
          ttl: 5m
```

**Trigger** — what causes the rule to fire. `posture_change` fires when a
route transitions to one of the listed postures. The trigger also fires for
new routes arriving with a matching posture.

**Cooldown** — minimum time between firings for the same prefix+peer
combination. Prevents alert storms during route flapping. Default: 60s.

**Actions** — what happens when the rule fires. Multiple actions can be
listed; they execute in order. If one action fails, subsequent actions
still run.

### Trigger Types

| Type | Description |
|---|---|
| `posture_change` | Route posture changes to one of the listed values |
| `new_route` | New route arrives with one of the listed postures |
| `cache_unhealthy` | RTR cache becomes unreachable |

### Recommended Rules

**Alert on hijacks (origin-invalid):**

```yaml
- name: "alert-hijack"
  trigger:
    type: posture_change
    postures: ["origin-invalid"]
  cooldown: 60s
  actions:
    - type: log
      level: warn
    - type: webhook
      url: "https://your-pagerduty-or-slack-endpoint"
```

**Alert on route leaks (path-suspect):**

```yaml
- name: "alert-route-leak"
  trigger:
    type: posture_change
    postures: ["path-suspect"]
  cooldown: 60s
  actions:
    - type: log
      level: warn
    - type: webhook
      url: "https://your-pagerduty-or-slack-endpoint"
```

**Flowspec mitigation on hijacks:**

```yaml
- name: "mitigate-hijack"
  trigger:
    type: posture_change
    postures: ["origin-invalid"]
  cooldown: 60s
  actions:
    - type: flowspec
      gobgp_address: "localhost:50051"
      dry_run: true
      ttl: 1h
      rule_action: drop
      max_rules: 50
```

---

## Webhooks

The webhook action sends an HTTP POST to your endpoint when a rule fires.

### Payload Format

```json
{
  "id": "b65b80a6-1234-5678-abcd-ef0123456789",
  "timestamp": "2026-05-06T14:23:01Z",
  "type": "posture_change",
  "prefix": "192.0.2.0/24",
  "peer_addr": "10.0.0.1",
  "peer_asn": 65000,
  "old_posture": "",
  "new_posture": "origin-invalid",
  "rov_state": "Invalid",
  "aspa_state": "Unknown",
  "router_id": "10.0.0.1"
}
```

### HMAC Signing

Set `secret` in the webhook config to enable HMAC-SHA256 signing. RAVEN
adds an `X-RAVEN-Signature` header to every request:
X-RAVEN-Signature: sha256=<hex-digest>

The signature is computed over the raw JSON body. Verify it on your
webhook receiver to confirm the request is from RAVEN.

```yaml
actions:
  - type: webhook
    url: "https://hooks.example.com/raven"
    secret: "your-shared-secret"
    max_attempts: 3
    timeout: 5s
```

### Retry Behaviour

Failed webhook deliveries are retried with exponential backoff up to
`max_attempts` times. If all attempts fail, the event is logged to the
dead-letter log at warn level. RAVEN never blocks the event pipeline on
webhook failures.

### Integrations

The webhook payload is compatible with:

- **PagerDuty** — use the Events API v2 with a custom transformer
- **Slack** — use an incoming webhook URL; transform the payload with a
  small proxy function
- **ServiceNow** — POST to the Table API endpoint for your incident table
- **Alertmanager** — use a webhook receiver with a custom template

---

## Flowspec Lifecycle

The Flowspec action automates the full mitigation lifecycle:
detect origin-invalid route
↓
generate Flowspec drop rule (dry-run)
↓
raven flowspec list     ← review
↓
raven flowspec toggle   ← inject live into GoBGP
↓
auto-expire after TTL   ← or toggle again to withdraw early

### Prerequisites

Flowspec injection requires a GoBGP instance with gRPC enabled. In the
demo lab, a GoBGP sidecar container handles this. In production, point
`gobgp_address` at your GoBGP instance.

```yaml
- type: flowspec
  gobgp_address: "your-gobgp:50051"
  dry_run: true
  ttl: 1h
  rule_action: drop
  max_rules: 50
  reaper_interval: 30s
```

### Dry-run Mode

Always start with `dry_run: true`. Rules are tracked in RAVEN's registry
and logged but not injected into GoBGP. This lets you validate that the
correct rules are being generated before enabling live injection.

```bash
# Review generated rules
raven flowspec list

# Example output
KEY                            ACTION  DRY-RUN  TTL        INSERTED
192.0.2.0/24|drop              drop    yes      4m50s      2026-05-06 14:23:01
```

### Live Injection

Toggle a specific rule from dry-run to live:

```bash
raven flowspec toggle "192.0.2.0/24|drop"
```

RAVEN calls GoBGP's AddPath API. The rule propagates to any BGP peers
configured on the GoBGP instance. Verify with:

```bash
docker exec gobgp gobgp global rib -a ipv4-flowspec
```

Toggle again to withdraw:

```bash
raven flowspec toggle "192.0.2.0/24|drop"
```

### TTL and Auto-expiry

Every rule has a TTL (default: 1 hour). The reaper goroutine runs every
`reaper_interval` and withdraws rules whose TTL has expired. This ensures
Flowspec rules don't persist indefinitely if RAVEN restarts or the
operator forgets to withdraw manually.

### Approval Webhook

For human-in-the-loop automation, configure an approval webhook. RAVEN
POSTs the proposed rule and waits for a 200 response before injecting:

```yaml
- type: flowspec
  gobgp_address: "localhost:50051"
  dry_run: false
  approval_webhook: "https://change-mgmt.example.com/approve"
  ttl: 1h
```

The approval endpoint receives the same JSON payload as a regular webhook
plus a `proposed_rule` field. Return 200 to approve, any other status to
reject.

!!! warning
    Max rules (`max_rules`) is a hard cap. When the registry is full, new
    rules are dropped and logged. Set this conservatively — a runaway
    event loop generating hundreds of Flowspec rules is worse than missing
    a few.

---

## raven audit

`raven audit` generates a read-only security posture report for any router
or peer in your BMP feed. It summarises ROV/ASPA coverage, flags top
offending ASNs, and provides actionable recommendations.

```bash
# Audit a specific router by peer address
raven audit --router 10.0.0.1

# JSON output for automation
raven audit --router 10.0.0.1 --format json

# Markdown output for incident reports
raven audit --router 10.0.0.1 --format markdown
```

### Example Output
Router: 10.0.0.1
Total Routes: 6
ROV Coverage:  83.3%
ASPA Coverage: 0.0%
POSTURE          COUNT
origin-only      3   (50%)
unverified       1   (17%)
origin-invalid   2   (33%)
TOP OFFENDERS
AS2121    1 route   origin-invalid   192.0.2.0/24
AS65001   1 route   origin-invalid   10.10.0.0/24
RECOMMENDATIONS

2 origin-invalid routes — run raven what-if --reject-invalid
ASPA coverage 0% — register ASPA objects at your RIR
Consider enabling reject-invalid after reviewing what-if output


### Automation

Pipe audit output to your ticketing or reporting pipeline:

```bash
# Weekly cron job
0 9 * * 1 raven audit --router 10.0.0.1 --format json \
  | curl -s -X POST https://your-api/security-report \
         -H "Content-Type: application/json" -d @-
```

---

## Warm-start Persistence

On clean shutdown, RAVEN saves the route table and RPKI cache to a
snapshot file. On restart, it restores from the snapshot before
reconnecting to BMP and RTR — eliminating the 30–90 second blind
window of a cold start.

Enable in config:

```yaml
snapshot:
  enabled: true
  path: "/var/lib/raven/snapshot.json"
  stale_timeout: 5m
```

Restored routes are marked stale. If BMP doesn't refresh a route within
`stale_timeout`, it is evicted. This prevents stale routes from persisting
if the topology changed during the downtime.

!!! note
    The snapshot uses JSON serialisation. When buf/protoc become available
    in your build environment, swap to protobuf for faster save/restore
    on large route tables. The interface is unchanged.
