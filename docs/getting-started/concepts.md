# Concepts

This page explains the core technologies RAVEN builds on. If you're already
familiar with BMP, RPKI, and ASPA, skip ahead to the
[Quick Start](quick-start.md).

## BMP — BGP Monitoring Protocol

BMP (RFC 7854) is a protocol that lets routers stream their BGP routing tables
to a monitoring station in real time. Think of it as a live feed of everything
your router is receiving from its BGP peers — before and after your import
policies are applied.

RAVEN acts as a BMP monitoring station. Your routers connect to RAVEN's BMP
listener and send a full table dump on startup, followed by incremental updates
as routes change. RAVEN sees every route your router receives, from every peer,
in real time.

**Why Adj-RIB-In Pre-Policy?**
RAVEN focuses on Adj-RIB-In Pre-Policy — what the router received before import
policy filtered it. This is where security analysis belongs. If a hijacked route
is being received and your import policy isn't filtering it, you want to know.

## RPKI — Resource Public Key Infrastructure

RPKI is a cryptographic framework that allows IP address holders to publish
signed records stating which ASNs are authorised to originate their prefixes.
These records are called ROAs (Route Origin Authorizations).

Validators like Routinator download and verify ROAs from the five RIRs (ARIN,
RIPE NCC, APNIC, LACNIC, AFRINIC) and make the validated data available to
routers and tools via the RTR protocol.

## ROV — Route Origin Validation

ROV (RFC 6811) is the process of checking a received BGP route against the RPKI
data. For each route, RAVEN checks:

- Does any ROA cover this prefix?
- If yes, does the ROA authorise the origin ASN?
- Is the prefix length within the ROA's maxLength?

The result is one of three states:

| State | Meaning |
|---|---|
| **Valid** | A ROA exists that authorises this origin ASN for this prefix |
| **Invalid** | A ROA exists but it does not authorise this origin ASN, or the prefix is too specific |
| **NotFound** | No ROA covers this prefix — no RPKI data available |

ROV catches **origin hijacks** — where an attacker announces your prefix from
their own ASN. If you have a ROA, the attacker's announcement will be Invalid.

**What ROV cannot catch:** Route leaks — where a legitimate ASN accidentally
re-announces routes they should not. The origin ASN is correct, so ROV passes.
This is where ASPA comes in.

## ASPA — AS Provider Authorization

ASPA (AS Provider Authorization) is a newer RPKI object type that lets an ASN
publish which other ASNs are its upstream providers. This creates a signed,
verifiable map of customer-provider relationships across the internet.

ARIN enabled ASPA object creation in January 2026. RIPE NCC in December 2025.
Adoption is under 1% of the global ASN space today — but growing.

RAVEN is the first operational tool that validates ASPA against your live
routing table via BMP.

## ASPA Path Verification

ASPA path verification (draft-ietf-sidrops-aspa-verification-24) walks the
AS_PATH of a received route and checks each hop:

- Does AS_i have an ASPA record?
- If yes, is AS_i-1 (the next hop toward you) listed as an authorised provider?

A route leak occurs when an AS re-announces routes in the wrong direction —
for example, a customer announcing a provider's routes to another provider.
This creates an AS_PATH where a hop is not an authorised customer-provider
relationship. ASPA catches this.

**Result states:**

| State | Meaning |
|---|---|
| **Valid** | Every hop in the AS_PATH is consistent with ASPA records |
| **Invalid** | At least one hop violates a published ASPA record — possible route leak |
| **Unknown** | Some ASes in the path have no ASPA records — cannot verify |
| **Unverifiable** | AS_PATH contains AS_SETs or other constructs that prevent verification |

## RTR — Router to Relying Party Protocol

RTR (RFC 8210, and RTR v2 in draft-ietf-sidrops-8210bis) is the protocol
validators use to deliver VRPs (Validated ROA Payloads) and ASPA data to
relying parties. RAVEN maintains a persistent RTR session to your validator
and keeps a local copy of all VRP and ASPA data, updated incrementally as
the validator's data changes.

RTR v2 adds ASPA PDU support. Routinator with `--enable-aspa` speaks RTR v2.

## The Security Posture Model

RAVEN combines ROV and ASPA results into a single security posture per route.
See [Security Postures](../user-guide/security-postures.md) for the full matrix
and operator guidance for each state.

## What RAVEN Is Not

- **Not a router** — RAVEN makes no changes to your routing table
- **Not an RPKI validator** — it connects to your existing validator via RTR
- **Not an external monitor** — it connects to your routers, not public route collectors
- **Not a replacement for BGPalerter** — BGPalerter watches from outside,
  RAVEN watches from inside. Run both.
