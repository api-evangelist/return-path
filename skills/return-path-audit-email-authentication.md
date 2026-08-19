---
name: Audit a domain's email authentication and DMARC posture
description: Put a sending domain under DMARC monitoring, then read its policy, mail origins, SPF and DKIM failures, and geolocation of the senders using it.
api: openapi/return-path-everest-api-openapi.yml
operations:
  - domainsPolicies
  - createDomain
  - domainOverview
  - domainMailOrigins
  - domainAuthenticationIssuesSPF
  - domainAuthenticationIssuesDKIM
  - domainSPFDomains
  - domainDKIMDomains
  - domainIPAddresses
  - domainGeolocation
  - domainDetailedReports
generated: '2026-08-13'
method: generated
source: openapi/return-path-everest-api-openapi.yml
---

# Audit a domain's email authentication and DMARC posture

The Everest Infrastructure module processes DMARC (RFC 7489) aggregate reports.
It tells a sender who is sending as their domain, whether that mail aligns under
SPF and DKIM, and what their published policy currently is.

## Before you start

- Base URL: `https://api.everest.validity.com/api`; `X-API-KEY` header on every
  request; datetimes UTC; 500 requests/minute with no rate-limit header.
- The path key for a domain is **the domain name itself**, not a surrogate id:
  `/2.0/infrastructure/domains/{domain}`.
- `createDomain` and `deleteDomain` are **not idempotent** — list first.

## Steps

1. **List what is already monitored.** `GET /2.0/infrastructure/domains`
   (`domainsPolicies`) returns each domain with `policy`, `volume`,
   `compliance`, `spf`, `dkim` and `reports`. `policy` is the DMARC policy
   Everest observes — `none`, `quarantine` or `reject`.
2. **Add the domain only if absent.** `POST /2.0/infrastructure/domains`
   (`createDomain`).
3. **Read the overview.** `GET /2.0/infrastructure/domains/{domain}`
   (`domainOverview`) — the compliance rate is the headline number: the share of
   observed mail that passes DMARC alignment.
4. **Find who is sending as the domain.**
   `GET /2.0/infrastructure/domains/{domain}/mailorigins`
   (`domainMailOrigins`) with `classification[]=compliant`,
   `classification[]=unauthenticated` and `classification[]=non-compliant`.
   Unauthenticated origins are either a forgotten legitimate sender (an ESP, a
   ticketing system, a payroll vendor) or a spoofer. Resolve every one of them
   before moving the policy to `quarantine` or `reject`.
5. **Diagnose the failures.**
   `GET /2.0/infrastructure/domains/{domain}/spfresults`
   (`domainAuthenticationIssuesSPF`) and
   `/dkimresults` (`domainAuthenticationIssuesDKIM`) give the failing checks.
   `/spfdomains` (`domainSPFDomains`) and `/dkimdomains` (`domainDKIMDomains`)
   list the domains seen in each mechanism — both accept `page` and `limit`.
6. **Corroborate.** `/ip` (`domainIPAddresses`) lists the sending IPs;
   `/geolocation` (`domainGeolocation`) accepts `country`, `region`, `page` and
   `limit` and is how obviously foreign spoofing shows itself;
   `/reports` (`domainDetailedReports`) returns the underlying aggregate reports.
7. **Close the loop.** Feed every sending IP found here into a reputation
   profile (see `return-path-monitor-sender-reputation.md`) so authentication and
   reputation are watched against the same list of identities.

## Handling failures

`401` no key, `403` module not entitled, `404` unknown domain or wrong version
segment, `429` over the 500/minute limit (exponential backoff — no `Retry-After`
is returned), `500` server-side. Errors are `{"status": "<free text>"}`; there is
no machine-readable error code.
