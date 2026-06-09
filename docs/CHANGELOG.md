# Changelog

All notable changes to RAVEN are recorded here.

## v0.3.1 (2026-06-09)

### Added
- IPv6 route monitoring via BMP (`MP_REACH_NLRI` / `MP_UNREACH_NLRI` parsing)
- IPv6 ROV validation
- IPv6 origin hijack scenario in demo lab (`./demo-master.sh hijack6`)

### Changed
- Demo lab internet router ASN changed from AS2121 to AS64496 to avoid
  conflicts with real-world RPKI records for AS2121
- Route leak scenario updated to use 145.102.136.0/22 (AS1199 / SURFnet)
  with real ASPA record (provider: AS1103) instead of synthetic prefix
