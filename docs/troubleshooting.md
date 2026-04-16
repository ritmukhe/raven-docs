# Troubleshooting

This page covers the most common issues encountered when deploying and running
RAVEN. These are hard-won lessons from real deployments.

## BMP Issues

### Routers Not Connecting

**Symptom:** `raven peers` shows no peers. `raven status` shows BMP listener
running but no sessions.

**Check 1 — Is the BMP listener actually running?**

```bash
ss -tlnp | grep 11019
```

If nothing shows, RAVEN is not listening. Check the logs:

```bash
journalctl -u raven -n 50
```

**Check 2 — Can the router reach RAVEN?**

From the router, test connectivity to RAVEN's BMP port:

```bash
# FRR — test from the router
ping 192.0.2.10
telnet 192.0.2.10 11019
```

**Check 3 — Firewall blocking the connection?**

```bash
# On the RAVEN host
iptables -L INPUT -n | grep 11019
```

**Check 4 — BMP config on the router correct?**

Verify the destination IP and port match RAVEN's listen address exactly.
A common mistake is configuring the router's management IP instead of the
IP reachable from the router's BGP interfaces.

---

### Routes Not Appearing

**Symptom:** `raven peers` shows a peer UP but `raven routes` returns nothing
or very few routes.

**Cause:** The BMP table dump is still in progress. After a router connects,
it sends a full table dump which can take 30-90 seconds for a full DFZ table.

**Check:**

```bash
# Watch route count grow
watch -n 2 raven status
```

The route count next to the peer should be increasing. Wait until it
stabilises before expecting complete results.

**If routes never appear:**

Check whether the router is configured to send Adj-RIB-In Pre-Policy.
Some router configurations only send Loc-RIB or post-policy by default:
FRR — explicitly configure pre-policy
bmp targets raven
monitor ipv4 unicast pre-policy
monitor ipv6 unicast pre-policy
exit

---

### All Routes Showing as Unverified

**Symptom:** Every route shows posture `unverified`. ROV state is `NotFound`
for all routes.

**Cause:** RAVEN has no VRPs — the RTR session to the validator is not working.

**Check:**

```bash
raven status
```

Look at the RTR CACHES section. If state is DOWN or last-sync is very old,
the RTR session has failed.

```bash
# Check Routinator is running
curl http://127.0.0.1:8323/api/v1/status
```

If Routinator is not running:

```bash
routinator server &
```

Wait for it to sync (up to 4 minutes cold start, ~13 seconds warm start).

---

## RTR / RPKI Issues

### Routinator Takes Too Long to Start

**Symptom:** RAVEN starts but RTR cache shows 0 VRPs for several minutes.

**Cause:** Routinator is doing a cold start and downloading the full RPKI
repository from scratch. This takes approximately 4 minutes.

**Solution:** Do not kill Routinator between sessions. A warm cache restart
takes about 13 seconds. If you must restart, wait for Routinator to fully
sync before starting RAVEN.

```bash
# Monitor Routinator sync progress
watch -n 5 'curl -s http://127.0.0.1:8323/api/v1/status | python3 -m json.tool'
```

Wait until `vrpsTotal` stabilises before proceeding.

---

### ASPA Validation Not Working

**Symptom:** All routes show ASPA state `Unknown`. `raven status` shows
`aspas=0` in the RTR cache.

**Cause:** One of:

1. Routinator is not started with `--enable-aspa`
2. RAVEN negotiated RTR v1 instead of v2
3. Routinator version does not support ASPA

**Check 1 — Routinator ASPA support:**

```bash
curl -s http://127.0.0.1:8323/api/v1/status | grep -i aspa
```

If no ASPA fields appear, restart Routinator with ASPA enabled:

```bash
pkill routinator
routinator server --enable-aspa &
```

**Check 2 — RTR version negotiation:**

```bash
raven status --format json | jq '.rtr_caches[] | {cache, rtr_version}'
```

If `rtr_version` shows `1`, RAVEN fell back to RTR v1. This happens when
Routinator was not ready with RTR v2 when RAVEN first connected.

**Fix:** Restart RAVEN after Routinator is fully synced and running with
`--enable-aspa`. RAVEN will renegotiate RTR v2 on reconnect.

!!! warning
    There is a known RTR v2 version negotiation timing issue. If RAVEN
    connects to Routinator before it is fully ready, it may permanently
    downgrade to RTR v1 for that session. Always ensure Routinator is
    fully synced before starting RAVEN.

---

### RTR Cache Keeps Going Stale

**Symptom:** `raven_rtr_cache_stale` fires repeatedly. RTR session drops
and reconnects.

**Check — Routinator memory:**

```bash
ps aux | grep routinator
```

If Routinator is using very high memory or CPU, it may be thrashing during
validation. Ensure the host has at least 2GB RAM available for Routinator.

**Check — Network connectivity:**

```bash
nc -zv 127.0.0.1 3323
```

If this fails, Routinator's RTR listener is not running. Check Routinator
logs and restart.

---

## Validation Issues

### Routes Showing as Origin-Invalid Unexpectedly

**Symptom:** Routes you expect to be valid are showing as `origin-invalid`.

**Cause:** Either a misconfigured ROA or a genuine issue.

**Investigate:**

```bash
raven routes --posture origin-invalid --format json | \
  jq '.[] | {prefix, origin_asn, rov_reason, matched_vrps}'
```

Look at `matched_vrps` — this shows which ROA covers the prefix and what
ASN it authorises. Compare against the route's `origin_asn`.

Common causes:

- ROA authorizes the wrong ASN (misconfiguration at the RIR)
- ROA maxLength is too restrictive — the route is a more-specific
- Legitimate route from a secondary origin ASN with no ROA

---

### What-If Results Seem Wrong

**Symptom:** `raven what-if --reject-invalid` shows 0 routes when you
expect some.

**Cause:** The route table may not be fully loaded yet, or ROV validation
has not completed.

**Check:**

```bash
raven status
```

Confirm route counts are stable and RTR cache has VRPs loaded. Then
re-run the what-if simulation.

---

## Demo Lab Issues

### Containerlab Subnet Conflict

**Symptom:** `containerlab deploy` fails with subnet conflict error.

**Fix:** Another Docker network is using `172.20.20.0/24`. Find and remove it:

```bash
docker network ls
docker network rm <conflicting-network-name>
```

---

### FRR Not Ready After Setup

**Symptom:** `demo-master.sh setup` warns that FRR did not become ready.

**Fix:** The BGP sessions need more time to converge. Wait 30 seconds and
check manually:

```bash
docker exec clab-raven-demo-upstream vtysh -c "show bgp summary"
```

Look for the `Established` state on all peers. If sessions are stuck in
`Active`, check the FRR configs and container connectivity.

---

### Prometheus Not Scraping RAVEN

**Symptom:** Grafana shows no data. Prometheus targets page shows RAVEN
as DOWN.

**Cause:** Common in WSL2 — Prometheus in Docker cannot reach RAVEN on
`localhost` because they are in different network namespaces.

**Fix:** Use the Docker bridge gateway IP instead of localhost:

```bash
ip route | grep default | awk '{print $3}'
# Typically 172.17.0.1
```

Update your Prometheus scrape config:

```yaml
static_configs:
  - targets: ["172.17.0.1:9595"]
```

Restart Prometheus after changing the config.

---

## Getting Help

If you encounter an issue not covered here:

1. Check the logs: `journalctl -u raven -n 100` or run with `--log-level debug`
2. Open an issue on [GitHub](https://github.com/nokia/bgp-routing-security-monitor/issues)
   with your RAVEN version (`raven version`), config file (redact IPs if needed),
   and the relevant log output
