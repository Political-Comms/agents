---
name: political-comms-api
description: Use when writing code that calls the Political Comms REST API, including authentication, creating and scheduling message projects, handling rate limits and errors, and verifying webhooks.
---

# Political Comms API Integration

Political Comms is the direct-to-carrier political texting platform. The REST API composes and schedules SMS, MMS, and RCS sends for campaigns, PACs, advocacy groups, fundraisers, and elected officials.

## When to use

Use this skill when the task involves sending compliant SMS, MMS, or RCS messages to United States voters or supporters on behalf of a political organization: GOTV reminders, fundraising asks, volunteer recruitment, event turnout, or survey outreach. The core workflow is create a project (POST /projects), send a test (POST /projects/{id}/test), then schedule it (POST /projects/{id}/schedule).

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
- Provisioning is human-initiated: a person generates keys at <https://app.politicalcomms.com/> under Admin, then API Keys. If no key exists, ask the operator to create one.
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

Schedule it:

```bash
curl https://api.politicalcomms.com/v1/projects/{project_id}/schedule \
  -X POST \
  -H "X-API-Key: $POLITICAL_COMMS_API_KEY" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{ "scheduled_at": "2026-11-03T18:00:00-05:00", "scheduled_timezone": "America/New_York" }'
```

`scheduled_at` must carry an explicit UTC offset; it may be now or in the past, in which case sending starts as soon as audience compilation finishes (no minimum lead time). `scheduled_timezone` must be one of the six supported US IANA zones: `America/New_York`, `America/Chicago`, `America/Denver`, `America/Los_Angeles`, `America/Anchorage`, or `Pacific/Honolulu`. `daily_cap_bypass` is optional (default `false`): brands T-Mobile meters (Aegis-vetted, non-political) carry a per-brand daily T-Mobile limit and a project otherwise pauses at it each Pacific day and must be started again to continue; set it to `true` to run the whole project through in one pass, accepting that messages to T-Mobile recipients over the limit may fail and are still billed.

Use an `Idempotency-Key` (a UUID per logical operation) on writes you might retry, always on schedule calls. Retries with the same key will not double-schedule.

## Rate limits

Per API key, over a 60-second sliding window: 100 requests per minute for reads, 60 per minute for writes, 30 per minute for deletes. Every response carries `X-RateLimit-*` headers with current usage and reset windows. Read the headers rather than counting requests. Back off before the ceiling; on a rate limit rejection, wait for the reset window before retrying.

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

## Email (early access)

The `/v1/email` surface covers sending domains, sender identities, lists and contacts, list imports, suppressions, campaigns, and templates.

**Every `/v1/email/*` endpoint returns `403 EMAIL_EARLY_ACCESS` until the email product reaches general availability.** That response is expected, not a bad key or a permissions problem. Do not retry it and do not report a credential failure. The contract is stable, so code written against it now keeps working once the flag is lifted.

What differs from the messaging surface:

- **Keyset pagination.** Email lists return `{ "data": [...], "has_more": bool, "next_cursor": string|null }` inside `data`, not a plain array. Page until `next_cursor` is null; cursors are opaque.
- **No inbox, no inbound email, no `email.opened`, no A/B testing.** Replies go to the sender identity's `reply_to` address.
- **DNS is manual.** `POST /v1/email/domains` returns the records to publish; poll until `status` is `active`.
- **Sender identities carry a read-only `gmail_verified_sender` object.** `{ status, submitted_at, verified_at }`, `status` one of `not_eligible | eligible | ready_to_submit | submitted | verified | suspended | rejected | expired`, standing in Google's Gmail Verified Sender Program via Campaign Verify. Never a sending gate.
- **Use suppressions to stop mailing someone.** `POST /v1/email/suppressions` survives a re-import; there is no public contact-delete.
- **Read `blocked` before scheduling.** `GET /v1/email/campaigns/{id}` names exactly what is stopping the schedule.
- **List import is one call.** `POST /v1/email/lists/import` fetches an HTTPS CSV you host, stages it, and commits, returning `202`; progress shows on the list in the dashboard. `mapping` is optional; a `400 VALIDATION_ERROR` carries `details.headers`, so send a mapping naming the email column instead of retrying the same body.
- **Read `lint` on template writes.** A template with lint errors saves but will not let a campaign schedule.
- **Setup and paid workflows are dashboard-only.** Registering domains and senders, list curation, validation, result exports, campaign pause/resume/test, and AI drafting are not in the API.

```bash
# Read a sending domain and the DNS records to publish yourself.
# Domains are registered in the dashboard; the API is read-only here.
curl https://api.politicalcomms.com/v1/email/domains \
  -H "X-API-Key: $POLITICAL_COMMS_API_KEY"
```

## Webhooks

Events: `message.sent`, `message.delivered`, `message.failed`, `message.replied`, `link.clicked`.

Email adds five more (early access, so they fire only once email is generally available): `email.delivered`, `email.bounced` (permanent bounces only), `email.complained`, `email.unsubscribed`, and `email.clicked` (unique, non-bot clicks). Each payload carries a stable `id` that is also the deduplication key.

Every delivery carries an HMAC-SHA256 signature in the `X-Webhook-Signature` header, formatted `sha256=...`. Compute HMAC-SHA256 over the raw request body with the webhook secret, compare with a constant-time comparison, and reject mismatches before reading the payload.

Respond 2xx quickly. Non-2xx triggers automatic retries, so make handlers idempotent on event delivery.

## References

- Full reference in one fetch: <https://politicalcomms.com/llms-full.txt>
- Index: <https://politicalcomms.com/llms.txt>
- Auth: <https://politicalcomms.com/auth.md>
