---
name: Diagnose an inbox placement drop
description: Work out why a sender's mail stopped reaching the inbox — read the latest inbox placement test, break it down per mailbox provider, and check the aggregate placement trend by region and by sending domain.
api: openapi/return-path-everest-api-openapi.yml
operations:
  - inboxTestsAllInboxTests
  - inboxTestsInboxTestByID
  - inboxTestsInboxTestAllProviders
  - inboxTestsInboxTestProviderByID
  - inboxTestsInboxTestHeadersByMailboxID
  - aggregateReportsInboxPlacementRegion
  - aggregateReportsInboxPlacementFromDomain
  - aggregateReportsMailboxProviderHealth
generated: '2026-08-13'
method: generated
source: openapi/return-path-everest-api-openapi.yml
---

# Diagnose an inbox placement drop

The Everest API is the Return Path platform's surviving surface. Inbox placement
is its original job: seed mailboxes across mailbox providers receive the sender's
campaign, and Everest reports where each copy landed.

## Before you start

- Base URL: `https://api.everest.validity.com/api`
- Auth: send `X-API-KEY: <key>` on **every** request. The key comes from Everest
  account settings. A missing or stale key returns `401` with
  `{"status":"Unauthorized: no valid API credentials provided."}`.
- All datetimes are UTC. Date filters use `startdate` / `enddate` as
  `YYYY-MM-DD HH:MM:SS`.
- Rate limit: 500 requests per minute per account. There is **no** rate-limit
  response header — on `429` back off exponentially, do not poll.
- **There is no idempotency contract.** Never blind-retry a `POST`/`PUT`/`DELETE`
  on a timeout; re-read the collection first and reconcile.

## Steps

1. **Find the test.** `GET /2.0/inbox/tests` (`inboxTestsAllInboxTests`) with
   `page` and `limit`. The response is `{"meta":{"total","limit","pages","params"},"results":[...]}`,
   so you can compute the last page instead of probing for a short one. Pick the
   test whose `subject` / `from` / `created` matches the campaign in question.
2. **Read the headline.** `GET /2.0/inbox/tests/{testId}`
   (`inboxTestsInboxTestByID`). `placement` is the inbox/spam/missing split and
   `authentication` is the SPF/DKIM/DMARC result for that send. If
   `authentication` is failing, stop here — that is the cause, and the fix lives
   in the Infrastructure module, not in content.
3. **Break it down by provider.** `GET /2.0/inbox/tests/{testId}/providers`
   (`inboxTestsInboxTestAllProviders`), then
   `GET /2.0/inbox/tests/{testId}/providers/{providerId}`
   (`inboxTestsInboxTestProviderByID`) for any provider whose placement is worse
   than the others. A drop concentrated at one provider is a reputation or
   filtering problem with that provider; a drop everywhere is authentication,
   content or list quality.
4. **Read the raw headers.** For a single seed mailbox,
   `GET /2.0/inbox/tests/{testId}/providers/{providerId}/headers/{headerId}`
   (`inboxTestsInboxTestHeadersByMailboxID`) returns what the receiving mailbox
   actually saw — filter verdicts, authentication results, routing.
   `GET /2.0/inbox/tests/{testId}/message` (`inboxTestsInboxTestFullMessage`)
   returns the full message as delivered.
5. **Check whether it is you or the provider.**
   `GET /2.0/inbox/health` (`aggregateReportsMailboxProviderHealth`) reports
   current mailbox-provider health. A platform-wide dip is not a sender problem.
6. **Confirm the trend.** `GET /2.0/inbox/aggregates/region`
   (`aggregateReportsInboxPlacementRegion`) and
   `GET /2.0/inbox/aggregates/fromdomain`
   (`aggregateReportsInboxPlacementFromDomain`) over a `startdate`/`enddate`
   window show whether this is one bad send or a sustained decline. Escalate a
   sustained decline into the reputation skill.

## Handling failures

| Status | Meaning | What to do |
|---|---|---|
| 401 | No valid API key | Re-read the key from account settings |
| 403 | Key lacks access to that Everest module | The account is not entitled — do not retry |
| 404 | Bad path, wrong version segment, or unknown id | Check `1.0` vs `2.0` and the id |
| 429 | 500 req/min exceeded | Exponential backoff; no `Retry-After` is sent |
| 500 | Server-side | Retry with backoff, then contact Validity support |

Every 4xx body is `{"status": "<free text>"}`. There is no machine-readable error
code, so branch on the HTTP status and log the `status` string verbatim.
