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
- **Email:** the `/v1/email` surface is generally available. Reads are open; writes need the email entitlement and return `403 ENTITLEMENT_REQUIRED` without it. See the Email section below.
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

Editing: `PATCH /projects/{id}` works on broadcast projects in `draft`, `awaiting_test`, `ready`, or `paused` (other statuses return `409 INVALID_STATE_TRANSITION` with `details.allowed`); only `organization_id` and `type` are immutable (`400 FIELD_NOT_EDITABLE`). `channel` (`10dlc`, `toll-free`, `short-code`), `brand_id`, `campaign_id`, and `toll_free_verification_id` are editable, but sending any of them requires the full setup for the effective channel (10dlc: `brand_id` and `campaign_id`; toll-free: `toll_free_verification_id`; short-code: none of the three) plus `phone_number_ids`, with the same checks and error codes as create; `409 CAMPAIGN_SUSPENDED` for a suspended campaign, `409 CAMPAIGN_EXPIRED` for an expired one, and `400 ASSET_OUT_OF_SCOPE` when the project's lists or media belong to the old brand or registration (send new `contact_list_ids`, `suppression_list_ids`, and `media_ids` in the same request). Any edit moves the project to `awaiting_test`. To resume a paused project after an edit: `PATCH`, then `POST /projects/{id}/test`, then `POST /projects/{id}/schedule`; recipients already sent are never re-sent. `POST /projects/{id}/schedule` returns `409 TOLL_FREE_NOT_VERIFIED` (`details.phoneNumberIds`) when a toll-free number has not finished verification.

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

`scheduled_at` must carry an explicit UTC offset; it may be now or in the past, in which case sending starts as soon as audience compilation finishes (no minimum lead time). `scheduled_timezone` must be one of the six supported US IANA zones: `America/New_York`, `America/Chicago`, `America/Denver`, `America/Los_Angeles`, `America/Anchorage`, or `Pacific/Honolulu`. `daily_cap_bypass` is optional (default `false`): for brands with a T-Mobile daily limit on file (political brands have none today), T-Mobile recipients over the limit otherwise send at a reduced rate; set it to `true` to skip the reduced rate and send them at full speed, accepting that a carrier rejection still pauses the project and rejected messages are still billed. It holds only until the next schedule or resume call rewrites it, so send it on every schedule or resume call. Scheduling also resumes a paused project.

Carrier throughput limits: T-Mobile caps messages per day per brand for brands that have a T-Mobile daily limit on file (political brands have none today), resetting at midnight Pacific. The cap is a speed limit, not a stop. Past it, T-Mobile and unknown-carrier recipients keep sending at about 300 message parts per minute per campaign while the platform watches receipts; after about 300 over-cap receipts with under 5% failures the rate rises to about 3,000 parts per minute. Other carriers are never slowed. Only the carrier pauses a project: if T-Mobile rejects messages over the cap (error code 016, or 20 or more failures that are at least half of the newest three minutes of T-Mobile receipts), the campaign's T-Mobile lane closes for the Pacific day and, once only T-Mobile and unknown-carrier recipients remain, the project auto-pauses (`GET /projects/{id}` returns `status: paused`, `pause_reason: brand_daily_cap`, `auto_paused: true`). Rejected messages are still billed. Run `POST /contact-lists/{id}/analyze` first and only T-Mobile recipients are affected. Resume with `POST /projects/{id}/schedule` (it accepts paused projects): schedule it for the next day during sending hours (the project's sending window: 8 AM to 10 PM in the time zone of most recipients, or 8 AM Eastern to 10 PM Pacific when no zone holds a majority) with a morning `scheduled_at` to wait for the reset, or call it without one to reopen the lane and try T-Mobile again at the reduced rate (it pauses again if T-Mobile is still rejecting). `daily_cap_bypass: true` skips the reduced rate and sends T-Mobile at full speed past the cap; a carrier rejection still pauses the project. Never resume at midnight. Every 10DLC campaign with an AT&T rate on file, political brands included, has its AT&T traffic delivered at that rate, counted in message parts (separate SMS and MMS rates; a two-part text counts twice). The platform sends AT&T recipients right away and the carrier queue delivers them at that rate, usually within about an hour; only in the last 90 minutes of the recipients' sending window does the platform hold back AT&T recipients that could not be delivered before the window closes. A campaign with no rate on file has no AT&T rate applied. AT&T never slows other carriers and never pauses a project. Check before scheduling with `GET /campaigns/{id}/throughput` (`t_mobile.daily_cap`, `used_today`, `remaining_today`, `att.sms_tpm`, `att.mms_tpm`). The lanes are independent: `t_mobile` is null when the brand has no T-Mobile daily limit on file or it has not synced yet, `att` is null only when the campaign has no AT&T rate on file (`sms_tpm` and `mms_tpm` can each be null), and `carrier_metered` is true if either lane applies (a political brand with an AT&T rate returns true with `t_mobile` null and `att` populated; false with both null when neither applies). `GET /projects/{id}/throughput` gives `t_mobile.estimated_recipients`, `att.estimated_minutes` (minutes AT&T needs at `att.tpm` for what the campaign already has waiting plus this project's AT&T recipients, accounting for the parts in the project's text), `carrier_coverage`; `t_mobile.will_pause` is always `false` and `t_mobile.estimated_send_days` is always `1` (kept for compatibility), results are cached up to 60 seconds, and a timed-out estimate returns 503 `CARRIER_ESTIMATE_TIMEOUT` (retry later). A null means not known right now, never zero. Poll for `paused`, not only `completed`: a `brand_daily_cap` pause means T-Mobile rejected messages over the cap, not that the project is stuck. Known `pause_reason` values: `brand_daily_cap` (T-Mobile rejected messages over the cap; schedule it for the next day's sending hours with a morning `scheduled_at`, or call schedule without one to try T-Mobile again at the reduced rate), `quiet_hours` (paused when the project's sending window closes: 10 PM in the time zone of most recipients, or 10 PM Pacific when no zone holds a majority; restart manually once the window opens), `carrier_block_rate`, `unregistered_campaign`, `insufficient_funds_auto_recharge_failed`, `insufficient_funds_ancestor`, `shared_phone_revoked`, `phone_released`, `organization_deleted`, `manual` or a short free-text reason (up to 100 characters) set by a user; other values may appear. Every auto-pause needs a manual restart via `POST /projects/{id}/schedule`. You can pause a project yourself with `POST /projects/{id}/pause` (from `active`, `sending`, or `scheduled`; optional `reason` up to 100 characters becomes `pause_reason`; already paused returns 200; other statuses return `409 INVALID_STATE_TRANSITION` with `details.allowed`), and resume it the same way, with `POST /projects/{id}/schedule`.

Contact lists: `POST /contact-lists/{id}/analyze` runs line-type analysis and is billable per lookup, charged to the list's organization. Send an `Idempotency-Key`; `402 INSUFFICIENT_BALANCE` queues nothing; `202` returns `cost_cents` and `numbers_queued`; `200` with `analysis.status` `complete` (cost 0) means nothing was left to analyze; a run already in progress returns `202` with cost 0. It is async: poll `GET /contact-lists/{id}` until `analysis.status` is `complete`, or subscribe to the opt-in `contact_list.analyzed` webhook (payload `id`, `list_id`, `name`, `analysis`, `analyzed_url`). `GET /contact-lists/{id}` returns `downloads.original_url` and `downloads.analyzed_url` (null until complete); `GET /contact-lists/{id}/download?type=original|analyzed` streams the CSV (`type` required; analyzed adds Phone Type, Carrier Name, Is Mobile, Is Opted Out, City, State; `409 ANALYSIS_NOT_COMPLETE` before completion; `404` outside the key's scope). The download URLs need your `X-API-Key` and do not expire. `POST /projects`, project update, and `POST /projects/{id}/schedule` return `409 LIST_ANALYSIS_IN_PROGRESS` while a contact list on the project is being analyzed; `details.lists` names each list. The request changes nothing: wait for `analysis.status` `complete` (or `contact_list.analyzed`), then retry it.

This call also returns `409 SENDING_PAUSED` if Political Comms has paused sending for the organization or platform-wide (`details.scope` is `organization` or `platform`). Do not retry; escalate to a human operator.

Use an `Idempotency-Key` (a UUID per logical operation) on writes you might retry, always on schedule calls. Retries with the same key will not double-schedule.

## Rate limits

600 requests per minute per API key, the same for every scope: bursts of up to 600 at once, refilling at 10 per second. Every response carries `X-RateLimit-*` headers with current usage. Read the headers rather than counting requests. Back off before the ceiling; on a rate limit rejection, wait the `Retry-After` seconds before retrying.

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

Every inbound text arrives on the `message.replied` webhook with `conversation_id`, `message_id`, `from`, `to`, `carrier` (the sender's mobile carrier, or null), and `text`. Answer it inside the same conversation, from the same number, with `POST /conversations/{conversation_id}/messages`:

```bash
curl https://api.politicalcomms.com/v1/conversations/{conversation_id}/messages \
  -X POST \
  -H "X-API-Key: $POLITICAL_COMMS_API_KEY" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{ "text": "Thanks for reaching out. Polls are open until 7pm." }'
```

Facts agents get wrong if they assume otherwise:

- The body is `{ "text": "...", "media_ids": ["<media id>"] }` and nothing else (`text` up to 10 SMS parts, 1,530 GSM-7 or 670 Unicode characters, or 1,500 characters with media; at most one `media_ids` entry, an id from `GET /media`; at least one of non-blank `text` or a media id is required; unknown properties and raw `media_urls` are a `400`). Without media the reply is SMS billed per segment; with media it is one MMS billed flat. The `202` data adds `media_urls`. The number is a property of the conversation, never of the request. The API never starts a conversation; a project send does.
- `202` means queued, not delivered. The outcome arrives on `message.sent`, `message.delivered`, or `message.failed` for the returned `message_id`; there is no separate reply event.
- Every refusal happens before any charge: `409 CONTACT_OPTED_OUT` (the contact replied STOP; do not retry), `409 CONVERSATION_NOT_SENDABLE`, `409 PROJECT_DELETED`, `409 PHONE_NUMBER_UNAVAILABLE`, `400 MEDIA_NOT_USABLE` (media not in the conversation's organization, not ready, or from a different brand than the project's), `409 SENDING_PAUSED` (sending paused for the organization or platform-wide; `details.scope` names which), `402 INSUFFICIENT_BALANCE`. A thread outside the key's organizations is `404 CONVERSATION_NOT_FOUND`, never `403`.
- `503 SEND_ENQUEUE_FAILED` means nothing was sent and nothing was charged: retry the same call. On any other `5xx`, read `GET /conversations/{conversation_id}/messages` and look for your text before retrying.
- Missed a webhook? `GET /conversations?updated_since=<iso>` lists threads with inbound messages, newest inbound first (default 7 days back, maximum 90; keyset paginated, page until `next_cursor` is null). Poll it at most once a minute; the webhook is the real-time path. `GET /conversations/{id}/messages` reads a thread newest first without marking it read.

## Triggered projects and campaigns

A triggered text project or email campaign has no contact list. Each person is added when its `trigger` fires and gets one message, right away or after a delay. The `trigger` has the same shape for both channels: `source` is `api`, `donation` (WinRed or Anedot gifts, filtered by `donor` `any|first_time|returning`, `gift` `any|one_time|recurring`, amount range, page names, source codes), or `event` (`text_link_click`, `text_reply`, `email_open`, `email_click`, `email_reply`, in `any` or the listed projects or campaigns); `delay_minutes` is 0 to 43200; `reentry` is `{ "mode": "once" }` or `{ "mode": "repeat", "min_gap_hours": 1..8760 }`.

1. Create it: `POST /v1/projects` with `"type": "triggered"` and a required `trigger` (no `contact_list_ids` or `suppression_list_ids`), or `POST /v1/email/campaigns` with a `trigger` instead of `list_ids` (not with `list_ids` or `sequence`).
2. Test it: `POST /v1/projects/{id}/test`, or the email test send. To rehearse a specific outcome without texting a real phone, send the test to a test phone number: any US number ending 555-0100 through 555-0199 (any area code). It never reaches a carrier, and the last two digits fix the result: 00 delivered; 01 failed 003 (blocked as spam); 02 failed 004 (rejected by destination); 03 failed 021 (not a mobile phone); 04 failed 012 (invalid number); 05 failed 300 (recipient opted out); 06 failed 005 (expired); 07 to 99 delivered. Webhooks fire as normal with carrier `Test carrier`, and the message is billed at the normal rate. See https://docs.politicalcomms.com/api-reference/testing.
3. Start it: `POST /v1/projects/{id}/start` (also resumes a paused project; `409 CONFLICT` if it is not triggered), or `POST /v1/email/campaigns/{id}/schedule` with no `scheduled_at`.
4. Add one recipient: `POST /v1/projects/{id}/recipients` with `{ "phone_number": "+12025550141", "fields": { "first_name": "Ada" }, "occurrence_id": "evt_123" }`, or `POST /v1/email/campaigns/{id}/recipients` with `email`. `202` queued with `send_at`; `200 already_entered` (`reason` `duplicate`, `already_entered`, `too_soon`); `409 CONTACT_OPTED_OUT` or `409 CONFLICT` (a paused or not started item refuses recipients, it does not queue them); `422 VALIDATION_ERROR` with `details { field, reason }`. Send an `occurrence_id` so a retry is a no-op.
5. Or hand out the hook URL: `POST /v1/projects/{id}/hook` or `POST /v1/email/campaigns/{id}/hook` returns `201 { hook_url, endpoint_url, secret }` once, and a new call retires the old ones. Prefer `endpoint_url` with the `secret` in an `X-Hook-Secret` header when the caller can set headers; `hook_url` carries the secret in its path for tools that cannot. `POST {hook_url}` with a JSON or form body (`phone_number` or `email`, optional `occurrence_id`, other scalar keys become fields) needs no API key and works from the generic webhook step in Zapier, Make, or n8n (there is no published Zapier app). Treat the URL as a secret.
6. End it: `POST /v1/projects/{id}/end` or `POST /v1/email/campaigns/{id}/end`. Final; people still waiting are not sent. Pause (`POST /v1/projects/{id}/pause`) is reversible.

A text due between 10 PM and 8 AM in the recipient's time zone is held until 8 AM; email has no hold. Opted-out numbers and suppressed addresses are never sent to. Fields are merge tags (`{first_name}`) and appear in reports, exports, and outbound webhook `contact.custom_fields`. `GET /v1/projects/stats` rows add `waiting_recipients` and `skipped_recipients`. Chain triggered items for follow-ups; there is no multi-step series.

## Email

The `/v1/email` surface covers sending domains, sender identities, suppressions (unsubscribes, bounces, complaints), campaigns, and templates.

**Write endpoints require the email entitlement on the organization and return `403 ENTITLEMENT_REQUIRED` without it.** Reads are open. That response is not a bad key. Do not retry it and do not report a credential failure; the organization needs email enabled on its account.

`POST /v1/email/campaigns/{id}/schedule` can additionally return `409 SENDING_PAUSED` when sending is paused. Do not retry it; escalate to a human operator.

What differs from the messaging surface:

- **Keyset pagination.** Email endpoints return `{ "data": [...], "has_more": bool, "next_cursor": string|null }` inside `data`, not a plain array. Page until `next_cursor` is null; cursors are opaque.
- **No inbox, no inbound email, no A/B testing.** `email.opened` exists but is approximate and opt-in. Replies go to the sender identity's `reply_to` address.
- **DNS is manual.** Domains are added in the dashboard (the Domains page), which shows the records to publish. Poll `GET /v1/email/domains/{id}` until `status` is `active`.
- **Sender identities carry a read-only `gmail_verified_sender` object.** `{ status, submitted_at, verified_at }`, `status` one of `not_eligible | eligible | ready_to_submit | submitted | verified | suspended | rejected | expired`, standing in Google's Gmail Verified Sender Program via Campaign Verify. Never a sending gate.
- **Use suppressions to stop mailing someone.** `POST /v1/email/suppressions` survives a re-import; there is no public contact-delete. Scope is `org`, `identity`, or `domain` and always applies. To hold people out of ONE campaign, put any list with emails in its `suppression_list_ids`.
- **Lists are one family for both channels.** `/v1/contact-lists` carries phones and emails; the `/v1/email/lists` endpoints are removed (404). Every list object has `channels` and an `email` object (status, counts, quality grade; null when the list has no emails). `GET /v1/contact-lists` defaults to `channel=text`, so ask for `channel=email` or `all` to find email lists. Emails reach a list only through `POST /v1/contact-lists/import` with an `email_column` (and `phone_column` for a file with both), which creates the list: there is no endpoint to create an empty list or to add or page through a list's emails. An import with an `email_column` needs the email entitlement of the list's organization. List ids are what `/v1/email/campaigns` takes in `list_ids` and `suppression_list_ids`.
- **Read `blocked` before scheduling.** `GET /v1/email/campaigns/{id}` names exactly what is stopping the schedule.
- **List import is one call, and the file IS the list.** `POST /v1/contact-lists/import` is the one list import, for phone numbers, emails, or both. It fetches an HTTPS CSV (50 MB cap, SSRF-guarded), checks the columns (each must be a header of the file, and its first rows must hold phone numbers or email addresses, else `400` and nothing is written), and commits it as a NEW list, answering `202` before any row is written. Send `list_name` and `phone_column`, `email_column`, or both; the email column is never guessed. `consent` (`source`, optional `note`) is optional and allowed only for an audience with an `email_column`; omit it and none is stored. `email_domain_id` scopes its emails to one sending domain (omit it for a list any campaign may use), and `acquired: true` (optional, default false) marks a purchased or rented list, which is sent more gradually. With `merge_tags`, phones map only the named columns while emails also map every remaining column by its default name (email custom field names match `^[a-z][a-z0-9_]{0,63}$`). Unknown top-level body fields are ignored; unknown keys inside `merge_tags` or `consent` return `400`. `email_column` needs the email entitlement (`403 ENTITLEMENT_REQUIRED`). Poll `GET /v1/contact-lists/{id}` for both sides: `import.status` for phones and `email.status` for emails, each until `ready` or `failed`. `purpose: "do_not_contact"` adds phones to an opt-out list and emails to unsubscribes (`GET /v1/email/suppressions`), and returns no `list_id`. A column that is not a header returns `400` with `details.headers`, so fix the column name rather than retrying the same body. Role addresses (info@, admin@) are always removed at import. `POST /v1/email/lists/import` was removed: send `name` as `list_name`, the `mapping` entry valued `email` as `email_column`, and the rest of `mapping` as `merge_tags`.
- **There is no validation step; sends are screened.** An address rated likely undeliverable is not sent to: message status `suppressed`, not a bounce, no `email.bounced` webhook, held out for 90 days, then re-checked. Campaigns send to every subscribed, unsuppressed contact. Yahoo-run mailboxes (Yahoo, AOL, AT&T, Verizon, Cox, Frontier) never join a list: imports remove them and count them in the list's `email.removed_at_upload` under `yahoo_family`. Recipients at providers whose mailboxes report spam at an unusually high rate (currently windstream.net) are removed the same way, counted under `high_complaint_domain`.
- **Pause and resume are on the API.** `POST /v1/email/campaigns/{id}/pause` (from `scheduled`, `compiling`, or `sending`; no body, any field is `400`; sets `status: paused`, `pause_reason: api`) and `POST /v1/email/campaigns/{id}/resume` (optional body `{ "resume_at": "<ISO 8601 date-time>" | null }`). A pause with reason `api`, `manual`, or `provider_refused` waits for a resume call (for `provider_refused`, after support fixes the cause); the other reasons (`insufficient_balance`, `insufficient_funds_ancestor`, `sender_inactive`, `no_content`, `tenant_paused`, `gmail_spam_rate`) resolve on their own once the cause clears. A resume while the cause persists pauses the campaign again. Already paused on pause, or already running on resume, returns `200` with the current state. A `resume_at` more than a minute ahead keeps the campaign `paused` and resumes it at that time (the campaign's `resume_at` field shows it; null otherwise; a later pause cancels it); a past `resume_at` is `400`. Resume checks sending controls (`409 SENDING_PAUSED`) and returns `409 EMAIL_CAMPAIGN_TEST_REQUIRED` when content changed during the pause and was not tested; send the test from the dashboard, then resume.
- **Read `lint` on template writes.** A template with lint errors saves but will not let a campaign schedule.
- **Setup and paid workflows are dashboard-only.** Registering domains and senders, list curation, result exports, campaign test sends, and AI drafting are not in the API. A triggered campaign is the exception: start it with the schedule call and no `scheduled_at`, end it with `/end`.

```bash
# Read a sending domain and the DNS records to publish yourself.
# Domains are registered in the dashboard; the API is read-only here.
curl https://api.politicalcomms.com/v1/email/domains \
  -H "X-API-Key: $POLITICAL_COMMS_API_KEY"
```

## Webhooks

Events: `message.sent`, `message.delivered`, `message.failed`, `message.replied`, `link.clicked`, plus the opt-in `contact_list.analyzed` (fires when a list finishes analysis; payload `id`, `list_id`, `name`, `analysis`, `analyzed_url`).

Email adds nine more: `email.sent`, `email.failed` (`reason` `rejected` or `render_failed`), `email.delivered`, `email.bounced` (permanent bounces only), `email.complained`, `email.unsubscribed`, `email.clicked` (unique, non-bot clicks), `email.replied`, and the opt-in `email.opened` (the first open; approximate, because mail apps that hide opens are not counted). Each payload carries a stable `id` that is also the deduplication key.

Every delivery carries an HMAC-SHA256 signature in the `X-Webhook-Signature` header, formatted `sha256=...`. Compute HMAC-SHA256 over the raw request body with the webhook secret, compare with a constant-time comparison, and reject mismatches before reading the payload.

Respond 2xx quickly. Non-2xx triggers automatic retries, so make handlers idempotent on event delivery. An endpoint that fails every delivery for 3 days is turned off and the organization's owners and admins are emailed; a human re-enables it in the dashboard after fixing the URL, and events from while it was off are not resent. To catch it first, poll `GET /webhook-endpoints` (read-only, the key's own organization): each endpoint reports `failing_since` (start of the current run of failed deliveries, null while healthy) and `disabled_reason` (`consecutive_failures` once it was turned off). The signing secret is never returned.

## References

- Full reference in one fetch: <https://politicalcomms.com/llms-full.txt>
- Index: <https://politicalcomms.com/llms.txt>
- Auth: <https://politicalcomms.com/auth.md>
