---
name: Set up and monitor sender reputation
description: Create a reputation profile for the sending IPs and domains, then watch Sender Score, blocklist listings and spam-trap hits against it.
api: openapi/return-path-everest-api-openapi.yml
operations:
  - profilesProfiles
  - profilesCreateProfile
  - profilesAddProfileItems
  - profilesProfileItems
  - profilesProfileByID
  - profilesProfileRDNSIssues
  - senderScoreIPDetail
  - blocklistsListings
  - spamTrapsSpamTrapsByProfile
  - spamTrapsSpamTrapsByDay
  - eventsDetailedEventsTraps
generated: '2026-08-13'
method: generated
source: openapi/return-path-everest-api-openapi.yml
---

# Set up and monitor sender reputation

Sender Score is the Return Path metric that survived the Validity acquisition
intact. In Everest it hangs off a **reputation profile** — a named bundle of the
IPs and domains a sender owns.

## Before you start

- Base URL: `https://api.everest.validity.com/api`; auth is `X-API-KEY` on every
  request; all datetimes are UTC; the limit is 500 requests/minute with no
  rate-limit header.
- Writes are **not idempotent**. Creating a profile twice creates two profiles.
  Always `GET` the collection first and match on `name`.

## Steps

1. **Check for an existing profile.** `GET /2.0/reputation/profiles`
   (`profilesProfiles`). Each result carries `id`, `name`, `items`, `enabled`,
   `issues`, `senderscore` and `blocklists` — often enough to answer the
   question without creating anything.
2. **Create one only if it is genuinely missing.**
   `POST /2.0/reputation/profiles` (`profilesCreateProfile`). Keep the returned
   `id`; there is no idempotency key to protect a retry.
3. **Add the sending identities.**
   `POST /2.0/reputation/profiles/{profileId}/items` (`profilesAddProfileItems`)
   with the IPs and domains to monitor. Verify with
   `GET /2.0/reputation/profiles/{profileId}/items` (`profilesProfileItems`)
   rather than trusting the write response.
4. **Read the reputation.** `GET /2.0/reputation/profiles/{profileId}`
   (`profilesProfileByID`) for the rollup, and
   `GET /2.0/reputation/senderscore/{ip}` (`senderScoreIPDetail`) for the
   per-IP Sender Score detail. Note the path key is the IP address itself.
5. **Check blocklists.** `GET /2.0/reputation/blocklists/listings`
   (`blocklistsListings`) returns current listings;
   `GET /2.0/reputation/blocklists` (`blocklistsBlocklists`) is the catalogue of
   lists Everest watches. A new listing is the single most common cause of a
   sudden placement collapse.
6. **Check spam traps.** `GET /2.0/reputation/traps/profile`
   (`spamTrapsSpamTrapsByProfile`) and `GET /2.0/reputation/traps/day`
   (`spamTrapsSpamTrapsByDay`) show the shape of the problem; the same slice
   exists by `ip`, `fromdomain`, `fromaddress`, `subject` and `friendlyfrom`.
   For the individual hits, `GET /2.0/reputation/events/traps`
   (`eventsDetailedEventsTraps`) accepts `startdate`, `enddate`, `traptype`,
   `ip`, `subject`, `fromdomain`, `fromaddress`, `friendlyfrom`, `profile_id`,
   `page` and `limit`. Trap type matters: **pristine** traps mean the list was
   bought or scraped; recycled traps mean the list is stale.
7. **Check rDNS.** `GET /2.0/reputation/profiles/{profileId}/rdns`
   (`profilesProfileRDNSIssues`) catches reverse-DNS misconfiguration, a cheap
   fix that mailbox providers weigh.

## Paging

Every listing endpoint takes `page` and `limit` and returns
`meta.total` / `meta.limit` / `meta.pages`. Read `meta.pages` once and walk it;
do not page until you get an empty result.

## Handling failures

`401` no key, `403` module not entitled, `404` bad id or wrong version segment,
`429` over 500 req/min (back off, no `Retry-After` is sent), `500` server-side.
The body is always `{"status": "<free text>"}` — log it, do not parse it.
