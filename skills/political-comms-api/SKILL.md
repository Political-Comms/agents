---
name: political-comms-api
description: Use when writing code that calls the Political Comms REST API, including authentication, creating and scheduling message projects, handling rate limits and errors, and verifying webhooks.
---

# Political Comms API Integration

Political Comms is the direct-to-carrier political texting platform. The REST API composes and schedules SMS, MMS, and RCS sends for campaigns, PACs, advocacy groups, fundraisers, and elected officials.

## When to use

Use this skill when the task involves sending compliant SMS, MMS, or RCS messages to United States voters or supporters on behalf of a political organization: GOTV reminders, fundraising asks, volunteer recruitment, event turnout, or survey outreach. The core workflow is create a project (POST /projects), send a test (POST /projects/{id}/test; each `test_contacts` entry may carry an optional `merge_values` object, tag name to value, so it renders without sampling a list contact), then schedule it (POST /projects/{id}/schedule).

Do not use Political Comms for commercial marketing outside politics, for messaging outside the United States, or for content that violates carrier political messaging rules. Documentation questions need no credentials: use the MCP server at <https://docs.politicalcomms.com/mcp>.

## Essentials

- **Base URL:** `https://api.politicalcomms.com/v1`
- **Source of truth:** the OpenAPI 3.1 spec at <https://politicalcomms.com/openapi.json> (mirror: <https://docs.politicalcomms.com/api-reference/openapi.json>). Always check it before writing a request. Never fabricate endpoints or fields.
- **Docs:** <https://docs.politicalcomms.com/api-reference/introduction>
- **Email:** the `/v1/email` surface is early access and returns `403 EMAIL_EARLY_ACCESS` until general availability. See the Email section below.
- **SDKs:** official TypeScript (`npm install @political-comms/sdk`) and Python (`pip install political-comms`) clients, a CLI (`npx @political-comms/cli`), and an MCP server (`npx -y @political-comms/mcp`). Direct HTTP against the spec works equally well.

## Authentication

Pass the API key in the `X-API-Key` header on every request:

```
X-API-Key: pc_live_1234567890abcdef
```

- Keys are prefixed `pc_live_` and are shown once at creation.
- Provisioning is human-initiated: a person generates keys at <https://app.politicalcomms.com/> under Admin → API. If no key exists, ask the operator to create one.
- Read keys from an environment variable or secret manager. Never hardcode, commit, or log a key.
- Agent-oriented walkthrough: <https://politicalcomms.com/auth.md>

## Quickstart: create, then schedule

A project is the unit of work: message body, optional media, link tracking, and the target contact list. Sending is two calls.

Create the project:

```bash
curl https://api.politicalcomms.com/v1/projects \
  -H "X-API-Key: $POLITICAL_COMMS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "organization_id": "org_01HX...",
    "name": "GOTV reminder - District 5",
    "protocol": "sms",
    "brand_id": "brand_01HX...",
    "campaign_id": "camp_01HX...",
    "phone_number_ids": ["pn_01HX..."],
    "contact_list_ids": ["list_01HX..."],
    "message_text": "Polls close at 7pm. Reply STOP to opt out."
  }'
```

`brand_id` and `campaign_id` are required on the default `10dlc` channel and omitted for `toll-free`. `contact_list_ids` is optional: omit it to create a draft and attach lists later via `PATCH /projects/{id}`. Validation is strict: unknown body properties are rejected with a 400.

This call, `POST /projects/{id}/copy`, and `POST /v1/email/campaigns` return `403 ONBOARDING_INCOMPLETE` once an organization's 14-day account setup grace window has passed with its business profile or funding step still incomplete (`details.missingSteps` names what's outstanding, `details.onboardingUrl` is `/onboarding`). Existing sends, schedules, and conversations are unaffected. Do not retry; escalate to a human operator, who completes setup in the dashboard.

`POST /projects` also returns `409 SENDING_PAUSED` if Political Comms has paused sending for the organization or platform-wide (`details.scope` is `organization` or `platform`). Do not retry; escalate to a human operator.

Schedule it:

```bash
curl https://api.politicalcomms.com/v1/projects/{project_id}/schedule \
  -X POST \
  -H "X-API-Key: $POLITICAL_COMMS_API_KEY" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{ "scheduled_at": "2026-11-03T18:00:00-05:00", "scheduled_timezone": "America/New_York" }'
```

`scheduled_at` must carry an explicit UTC offset; it may be now or in the past, in which case sending starts as soon as audience compilation finishes (no minimum lead time). `scheduled_timezone` must be one of the six supported US IANA zones: `America/New_York`, `America/Chicago`, `America/Denver`, `America/Los_Angeles`, `America/Anchorage`, or `Pacific/Honolulu`. `daily_cap_bypass` is optional (default `false`): brands T-Mobile meters (Aegis-vetted, non-political) carry a per-brand daily T-Mobile limit and a project otherwise pauses at it each Pacific day and must be started again to continue; set it to `true` to run the whole project through in one pass, accepting that messages to T-Mobile recipients over the limit may fail and are still billed. It holds only until the next schedule or resume call rewrites it, so send it on every schedule or resume call. Scheduling also resumes a paused project.

Carrier throughput limits: T-Mobile caps messages per day per brand for Aegis-vetted (non-political) 10DLC brands, resetting at midnight Pacific. At the cap, recipients on known other carriers keep sending; T-Mobile and unknown-carrier recipients wait, and when only those remain the project auto-pauses (`GET /projects/{id}` returns `status: paused`, `pause_reason: brand_daily_cap`, `auto_paused: true`). Run `POST /contact-lists/{id}/analyze` first and only T-Mobile recipients wait. Resume with `POST /projects/{id}/schedule` (it accepts paused projects): schedule it for the next day during sending hours (8 AM to 10 PM recipients' local time) with a morning `scheduled_at`, or resume now with `daily_cap_bypass: true` (over-cap T-Mobile may fail, still billed). Never resume at midnight. AT&T paces per minute per campaign (separate SMS and MMS rates) and never pauses, it only slows. Political (Campaign Verify) brands have neither limit. Check before scheduling with `GET /campaigns/{id}/throughput` (`t_mobile.daily_cap`, `used_today`, `remaining_today`, `att.sms_tpm`, `att.mms_tpm`). The lanes are independent: `t_mobile` is null until a T-Mobile daily cap has synced, `att` is null unless the brand is Aegis-vetted and an AT&T tier is known (`sms_tpm` and `mms_tpm` can each be null), and `carrier_metered` is true if either lane applies (false with both null for political brands). `GET /projects/{id}/throughput` gives `t_mobile.will_pause`, `t_mobile.estimated_send_days`, `att.estimated_minutes`, `carrier_coverage`; with `daily_cap_bypass` on it reports `will_pause: false` and `estimated_send_days: 1`, results are cached up to 60 seconds, and a timed-out estimate returns 503 `CARRIER_ESTIMATE_TIMEOUT` (retry later). A null means not known right now, never zero (if `used_today` is null, `will_pause` is false). Poll for `paused`, not only `completed`: a `brand_daily_cap` pause is waiting for the next Pacific day, not stuck. Known `pause_reason` values: `brand_daily_cap` (schedule it for the next day's sending hours with a morning `scheduled_at`, or resume now with `daily_cap_bypass: true`), `quiet_hours` (paused at 10 PM recipients' local time; restart manually next morning), `carrier_block_rate`, `unregistered_campaign`, `provider_error`, `insufficient_funds_auto_recharge_failed`, `insufficient_funds_ancestor`, `shared_phone_revoked`, `phone_released`, `organization_deleted`, `manual` or a short free-text reason (up to 100 characters) set by a user; other values may appear. Every auto-pause needs a manual restart via `POST /projects/{id}/schedule`.

Contact lists: `POST /contact-lists/{id}/analyze` runs line-type analysis and is billable per lookup, charged to the list's organization. Send an `Idempotency-Key`; `402 INSUFFICIENT_BALANCE` queues nothing; `202` returns `cost_cents` and `numbers_queued`; `200` with `analysis.status` `complete` (cost 0) means nothing was left to analyze; a run already in progress returns `202` with cost 0. It is async: poll `GET /contact-lists/{id}` until `analysis.status` is `complete`, or subscribe to the opt-in `contact_list.analyzed` webhook (payload `id`, `list_id`, `name`, `analysis`, `analyzed_url`). `GET /contact-lists/{id}` returns `downloads.original_url` and `downloads.analyzed_url` (null until complete); `GET /contact-lists/{id}/download?type=original|analyzed` streams the CSV (`type` required; analyzed adds Phone Type, Carrier Name, Is Mobile, Is Opted Out, City, State; `409 ANALYSIS_NOT_COMPLETE` before completion; `404` outside the key's scope). The download URLs need your `X-API-Key` and do not expire.

This call also returns `409 SENDING_PAUSED` if Political Comms has paused sending for the organization or platform-wide (`details.scope` is `organization` or `platform`). Do not retry; escalate to a human operator.

Use an `Idempotency-Key` (a UUID per logical operation) on writes you might retry, always on schedule calls. Retries with the same key will not double-schedule.

## Rate limits

600 requests per minute per API key, the same for every scope: bursts of up to 600 at once, refilling at 10 per second. Every response carries `X-RateLimit-*` headers with current usage and reset windows. Read the headers rather than counting requests. Back off before the ceiling; on a rate limit rejection, wait for the reset window before retrying.

## Errors

Structured JSON on every error:

```json
{
  "success": false,
  "error": "Human-readable message",
  "code": "machine_readable_code",
  "statusCode": 400
}
```

- Branch on `code` and `statusCode`, not the error string.
- 4xx (other than rate limiting): fix the request, do not retry the same payload.
- 5xx: retry with exponential backoff and an `Idempotency-Key`.

## Replying to inbound texts

Every inbound text arrives on the `message.replied` webhook with `conversation_id`, `message_id`, `from`, `to`, and `text`. Answer it inside the same conversation, from the same number, with `POST /conversations/{conversation_id}/messages`:

```bash
curl https://api.politicalcomms.com/v1/conversations/{conversation_id}/messages \
  -X POST \
  -H "X-API-Key: $POLITICAL_COMMS_API_KEY" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{ "text": "Thanks for reaching out. Polls are open until 7pm." }'
```

Facts agents get wrong if they assume otherwise:

- The body is `{ "text": "..." }` and nothing else (SMS only, up to 1,600 characters; unknown properties are a `400`). The number is a property of the conversation, never of the request. The API never starts a conversation; a project send does.
- `202` means queued, not delivered. The outcome arrives on `message.sent`, `message.delivered`, or `message.failed` for the returned `message_id`; there is no separate reply event.
- Every refusal happens before any charge: `409 CONTACT_OPTED_OUT` (the contact replied STOP; do not retry), `409 CONVERSATION_NOT_SENDABLE`, `409 PROJECT_DELETED`, `409 PHONE_NUMBER_UNAVAILABLE`, `409 SENDING_PAUSED` (sending paused for the organization or platform-wide; `details.scope` names which), `402 INSUFFICIENT_BALANCE`. A thread outside the key's organizations is `404 CONVERSATION_NOT_FOUND`, never `403`.
- `503 SEND_ENQUEUE_FAILED` means nothing was sent and nothing was charged: retry the same call. On any other `5xx`, read `GET /conversations/{conversation_id}/messages` and look for your text before retrying.
- Missed a webhook? `GET /conversations?updated_since=<iso>` lists threads with inbound messages, newest inbound first (default 7 days back, maximum 90; keyset paginated, page until `next_cursor` is null). Poll it at most once a minute; the webhook is the real-time path. `GET /conversations/{id}/messages` reads a thread newest first without marking it read.

## Email (early access)

The `/v1/email` surface covers sending domains, sender identities, lists and contacts, list imports, suppressions, campaigns, and templates.

**Every `/v1/email/*` endpoint returns `403 EMAIL_EARLY_ACCESS` until the email product reaches general availability.** That response is expected, not a bad key or a permissions problem. Do not retry it and do not report a credential failure. The contract is stable, so code written against it now keeps working once the flag is lifted.

**Once GA, every write under `/v1/email/*` also requires the `email` entitlement.** Without it, the write returns `403 ENTITLEMENT_REQUIRED` with `details.entitlement: "email"`; `POST /v1/email/campaigns/{id}/schedule` additionally returns `409 SENDING_PAUSED` when sending is paused. Do not retry either; escalate to a human operator.

What differs from the messaging surface:

- **Keyset pagination.** Email lists return `{ "data": [...], "has_more": bool, "next_cursor": string|null }` inside `data`, not a plain array. Page until `next_cursor` is null; cursors are opaque.
- **No inbox, no inbound email, no `email.opened`, no A/B testing.** Replies go to the sender identity's `reply_to` address.
- **DNS is manual.** `POST /v1/email/domains` returns the records to publish; poll until `status` is `active`.
- **Sender identities carry a read-only `gmail_verified_sender` object.** `{ status, submitted_at, verified_at }`, `status` one of `not_eligible | eligible | ready_to_submit | submitted | verified | suspended | rejected | expired`, standing in Google's Gmail Verified Sender Program via Campaign Verify. Never a sending gate.
- **Use suppressions to stop mailing someone.** `POST /v1/email/suppressions` survives a re-import; there is no public contact-delete. Scope is `org`, `identity`, or `domain` and always applies. To hold people out of ONE campaign, put an ordinary email list in its `suppression_list_ids`.
- **Read `blocked` before scheduling.** `GET /v1/email/campaigns/{id}` names exactly what is stopping the schedule.
- **List import is one call, and the file IS the list.** `POST /v1/email/lists/import` fetches an HTTPS CSV you host, stages it, and commits it as a NEW list, returning `202`; progress shows on the list in the dashboard. There is no `email_list_id`: `name` defaults to the file name and `email_domain_id` scopes the list to one sending domain. `mapping` is optional; a `400 VALIDATION_ERROR` carries `details.headers`, so send a mapping naming the email column instead of retrying the same body.
- **Read `lint` on template writes.** A template with lint errors saves but will not let a campaign schedule.
- **Setup and paid workflows are dashboard-only.** Registering domains and senders, list curation, validation, result exports, campaign pause/resume/test, and AI drafting are not in the API.

```bash
# Read a sending domain and the DNS records to publish yourself.
# Domains are registered in the dashboard; the API is read-only here.
curl https://api.politicalcomms.com/v1/email/domains \
  -H "X-API-Key: $POLITICAL_COMMS_API_KEY"
```

## Webhooks

Events: `message.sent`, `message.delivered`, `message.failed`, `message.replied`, `link.clicked`, plus the opt-in `contact_list.analyzed` (fires when a list finishes analysis; payload `id`, `list_id`, `name`, `analysis`, `analyzed_url`).

Email adds five more (early access, so they fire only once email is generally available): `email.delivered`, `email.bounced` (permanent bounces only), `email.complained`, `email.unsubscribed`, and `email.clicked` (unique, non-bot clicks). Each payload carries a stable `id` that is also the deduplication key.

Every delivery carries an HMAC-SHA256 signature in the `X-Webhook-Signature` header, formatted `sha256=...`. Compute HMAC-SHA256 over the raw request body with the webhook secret, compare with a constant-time comparison, and reject mismatches before reading the payload.

Respond 2xx quickly. Non-2xx triggers automatic retries, so make handlers idempotent on event delivery.

## References

- Full reference in one fetch: <https://politicalcomms.com/llms-full.txt>
- Index: <https://politicalcomms.com/llms.txt>
- Auth: <https://politicalcomms.com/auth.md>
