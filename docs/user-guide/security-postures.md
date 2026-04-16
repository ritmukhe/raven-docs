# Security Postures

RAVEN combines ROV and ASPA validation results into a single security posture
per route. This page explains what each posture means and what to do about it.

## The Posture Matrix

| ROV State | ASPA State | Posture | Meaning |
|---|---|---|---|
| Valid | Valid | **secured** | Origin and path both verified |
| Valid | Unknown | **origin-only** | Origin verified, path not verifiable |
| Valid | Invalid | **path-suspect** | Origin legitimate but path violates ASPA |
| NotFound | Valid | **path-only** | Path clean but no ROA for origin |
| NotFound | Unknown | **unverified** | No RPKI coverage at all |
| NotFound | Invalid | **path-suspect** | No origin coverage and path violates ASPA |
| Invalid | Any | **origin-invalid** | Origin fails ROV |

## Posture Details

### secured

The origin ASN is authorised by a ROA and every hop in the AS_PATH is
consistent with published ASPA records. This is the highest confidence posture.

**What to do:** Nothing. This is the target state.

**How common today:** Rare — requires both a ROA and ASPA objects for every
AS in the path. Will become more common as ASPA adoption grows.

---

### origin-only

The origin ASN is authorised by a ROA but the AS_PATH cannot be fully verified
because some ASes in the path have no ASPA records.

**What to do:** Nothing urgent. This is the expected state for most RPKI-covered
routes during the current early phase of ASPA deployment. You are protected
against origin hijacks but not route leaks.

**How common today:** This will be the most common posture for routes with ROAs
until ASPA adoption increases significantly.

---

### path-suspect

At least one hop in the AS_PATH violates a published ASPA record. This means
an AS that has published its provider relationships is routing through an AS
that is not listed as its provider — a strong indicator of a route leak.

**What to do:** Investigate immediately.

- Run `raven routes --posture path-suspect --format json` to see the full
  AS_PATH and the specific failing hop
- Check whether the route is being used (is it the best path?)
- Contact your upstream if the failing hop is in their path
- Consider filtering the route if you have confirmed it is a leak

**How common today:** Uncommon — requires ASPA objects to exist for the
failing AS. Will become more detectable as ASPA adoption grows.

!!! warning
    A path-suspect route does not mean your network is under attack. Route
    leaks are usually accidental misconfigurations, not malicious. Investigate
    before taking action.

---

### path-only

The AS_PATH looks clean according to ASPA records but the origin AS has no ROA.
The path is verifiable but origin legitimacy cannot be confirmed.

**What to do:** Encourage the origin AS to register ROAs with their RIR.
You can use `raven aspa recommend` to identify ASes that could improve their
RPKI coverage.

---

### unverified

No RPKI coverage at all — no ROA and no ASPA records for any AS in the path.
This is the status quo for the majority of the internet today.

**What to do:** Nothing urgent for individual routes. At a network level,
track whether your upstream providers and peers are improving their RPKI
coverage over time. Use `raven what-if --reject-invalid` to understand what
deploying ROV would mean for your network.

**How common today:** This will be the largest bucket in most production
networks.

---

### origin-invalid

A ROA exists for this prefix but it does not authorise the origin ASN
announcing it. This is either:

- An **origin hijack** — a malicious AS announcing your prefix
- A **ROA misconfiguration** — your own ROA is wrong (wrong ASN or maxLength)
- A **legitimate announcement** that predates the ROA — rare but possible

**What to do:**

1. Check the matched VRPs: `raven routes --posture origin-invalid --format json`
2. Is the origin ASN one you recognise? If not, it may be a hijack
3. Is it your own prefix? Check your ROA configuration at your RIR
4. If it is a confirmed hijack, contact your upstream and consider filtering

!!! danger
    A sudden increase in origin-invalid routes for your own prefixes is a
    strong indicator of an active origin hijack. Use `raven watch --posture
    origin-invalid` to be alerted the moment it happens.

## Using Postures Operationally

### Daily Operations

```bash
raven watch --posture origin-invalid
raven watch --posture path-suspect
```

### Incident Investigation

```bash
raven routes --posture origin-invalid
raven routes --posture path-suspect
raven routes --prefix 192.0.2.0/24
raven routes --posture origin-invalid --format json | jq '.[] | {prefix, peer, origin}'
```

### Policy Planning

```bash
raven what-if --reject-invalid
raven what-if --aspa-enforce
raven aspa recommend --min-observations 100
```
