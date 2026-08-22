---
name: goodera-book-a-volunteering-event
description: Find a Goodera volunteering opportunity and book it as a real event for a team, then manage who is enrolled.
api: goodera-developer-api
generated: '2026-08-22'
method: generated
source: https://www.goodera.com/resources/api
operations:
  - GET /master-data/country
  - GET /master-data/timezone
  - GET /master-data/language
  - GET /opportunities
  - GET /opportunities/{id}
  - POST /events
  - GET /events/{eventId}
  - POST /registrations
  - GET /registrations/by_event/{eventId}
  - DELETE /registrations/{registrationId}
---

# Book a Goodera volunteering event

Every operation below is documented at <https://www.goodera.com/resources/api>. Goodera
publishes no OpenAPI document, so there are no operationIds — the HTTP method and path
above are the contract as published.

## Before you start

- Base URL is `https://developer-api.goodera.com`.
- Send `x-api-key: <YOUR-API-KEY>` on every call except `/master-data/*`, which is
  documented as unauthenticated.
- Keys are not self-service. They are issued by Goodera through the "Request access"
  form on the reference page. There is **no sandbox and no test mode** — every call in
  this skill hits production and creates real bookings with real nonprofits.

## Step 1 — Load the reference lists first

`GET /master-data/timezone`, `GET /master-data/country`, `GET /master-data/language`.

Do this before anything else. `POST /events` requires a `timezone` value drawn from
Goodera's own timezone master list, not an arbitrary IANA name, and `country` and
`language` are validated the same way. No `400`/`422` shape is documented, so a bad
value will fail in an unspecified manner — pick from the list rather than guessing.

## Step 2 — Find an opportunity

`GET /opportunities?pageSize=10&page=1`, optionally filtered by `countryCode`,
`durations` (minutes, e.g. `[60,90]`) and `eventLocationType` (`VIRTUAL` | `IN_PERSON`).

The response envelope is `{ rows, total, page, pageSize, totalPages }`. Page with
`page`/`pageSize`. No maximum `pageSize` is published, so do not assume a large value is
honoured — read `pageSize` back off the response to see what the server actually applied.

`GET /opportunities/{id}` returns the single record. Check `isPaid` before booking:
opportunities carry that flag and this skill does not cover commercial terms.

## Step 3 — Create the event — THIS IS THE POINT OF NO RETURN

`POST /events` with `opportunityId`, `startTimeStamp`, `endTimeStamp` (both ISO 8601),
`timezone`, `country`, `language`, `email`, `name`, `expectedVolunteerCount`, and for
in-person formats `eventLocation` (city, country, coordinates, address) plus optional
`addressLine`.

**Stop and confirm with a human before you call this.** Two reasons, both verified
against Goodera's published documentation:

1. **There is no documented way to cancel or delete an event.** No cancel, delete or
   reschedule operation appears in the reference. The platform clearly has the concept —
   `cancelled` is a real event state — but no partner-callable operation to reach it is
   published. Treat this write as irreversible.
2. **There is no idempotency key.** No idempotency header or replay-safety mechanism is
   documented. If this request times out you cannot safely retry it, because a retry may
   create a second real event. On a timeout, do **not** retry — fetch and reconcile
   instead.

## Step 4 — Verify what you created

`GET /events/{eventId}`.

Always do this after a `POST /events` that timed out or returned an ambiguous status. It
is the only way to distinguish "created but the response was lost" from "not created",
and it is the substitute for the idempotency guarantee the API does not offer.

## Step 5 — Enrol volunteers

`POST /registrations` with `eventId` and `email`. One call per volunteer.

This is the one write on the API you can take back: `DELETE /registrations/{registrationId}`
removes an enrolment. Note the caveat — **no window is stated**. The documentation does
not say whether deletion still works once the event starts, once attendance is recorded,
or once the event completes. Do not promise a volunteer they can be removed later.

`GET /registrations/by_event/{eventId}` lists who is enrolled. Use it to reconcile rather
than tracking state locally, and use it before re-adding anyone, since duplicate-enrolment
behaviour is not documented.

## Error handling — read this carefully, Goodera's codes are non-standard

The published status table for every operation is `200`, `401`, `403`, `500`. Two traps:

- **`401` and `403` are inverted relative to RFC 9110.** Goodera documents `401` as
  "Unauthorized" and `403` as "Unauthenticated". Standard client logic will branch the
  wrong way. Treat `403` as "no valid credential" and `401` as "credential present but
  not permitted", and log both verbatim.
- **`500` does not reliably mean "server broken".** The service returns HTTP `500` for
  routing misses, nesting the real status in `data.statusCode`. Before retrying a `500`,
  parse `data.statusCode`; if it is `404` you called a path that does not exist and a
  retry will never succeed.

There is no `429` in the contract, no rate-limit headers, no `Retry-After`, no request-id
header to quote in a support ticket, and no status page. If calls start failing, you have
no published signal to distinguish throttling from an outage.

## What this skill deliberately does not do

`POST /participation` records volunteer hours, and those hours feed corporate CSR and ESG
reporting. **No correction, void or amend operation is documented.** Recording a wrong hour
is not undoable by any published means, so this skill leaves it to a human.
