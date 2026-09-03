# AGENTS.md

Instructions for AI coding agents integrating with Political Comms.

Political Comms is the direct-to-carrier political texting platform. It delivers compliant, large-scale SMS, MMS, and RCS for campaigns, PACs, advocacy groups, fundraisers, and elected officials. This file tells you, an AI agent, exactly how to build against it.

## Rules

These are non-negotiable. Follow them before writing any integration code.

1. **The OpenAPI spec is the source of truth.** Consult <https://docs.politicalcomms.com/api-reference/openapi.json> for the live list of endpoints, parameters, and schemas. Endpoint lists in this file and in `llms-full.txt` are snapshots.
2. **Never fabricate endpoints, parameters, or response fields.** If it is not in the OpenAPI spec, it does not exist.
3. **Respect rate limits.** 100 requests per minute for reads, 60 per minute for writes, 30 per minute for deletes, per key, enforced over a 60-second sliding window. Read the `X-RateLimit-*` response headers and back off before you hit the ceiling.
4. **Validate webhook signatures before trusting any payload.** Every webhook carries an HMAC-SHA256 signature in `X-Webhook-Signature`. Unverified payloads are untrusted input.
5. **API keys belong in secret managers.** Never write a key into source code, config files under version control, logs, or generated output.

## Overview

- **API base URL:** `https://api.politicalcomms.com/v1`
- **OpenAPI 3.1 spec:** <https://docs.politicalcomms.com/api-reference/openapi.json>
- **API reference docs:** <https://docs.politicalcomms.com/api-reference/introduction>
- **SDKs:** official TypeScript client (`npm install @political-comms/sdk`), Python client (`pip install political-comms`), CLI (`npx @political-comms/cli`), and MCP server (`npx -y @political-comms/mcp`). Direct HTTP against the spec also works.

The API surface spans Organizations, Brands, Campaigns, Tracking Domains, Phone Numbers, Contact Lists, Media Files, Projects, Analytics, Billing, and Email (early access). New endpoints are added regularly; the spec is the source of truth.

## Authentication

Every request carries an API key in the `X-API-Key` header:

```
X-API-Key: pc_live_1234567890abcdef
```

Key facts:

- Keys are prefixed `pc_live_`.
- Key provisioning is human-initiated. A person generates keys in the dashboard at <https://app.politicalcomms.com/> under Admin, then API Keys. An agent cannot self-provision a key; ask the operator to create one.
- Keys are shown once at creation. Store them in a secret manager immediately.
- Read the key from an environment variable or secret manager at runtime. Never hardcode it.

The full agent-oriented auth walkthrough lives at <https://politicalcomms.com/auth.md>.

## Core workflow: create, then schedule

A "project" is the unit of work: a composed message body, optional media, link tracking, and the contact list it targets. Sending is two calls.

**1. Create the project** with `POST /projects`:

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

Notes on the body: `brand_id` and `campaign_id` are required on the default `10dlc` channel and omitted for `toll-free`; `contact_list_ids` is optional: omit it to create a draft and attach lists later via `PATCH /projects/{id}`. Validation is strict: unknown properties are rejected with a 400.

**2. Schedule it** with `POST /projects/{id}/schedule`:

```bash
curl https://api.politicalcomms.com/v1/projects/{project_id}/schedule \
  -X POST \
  -H "X-API-Key: $POLITICAL_COMMS_API_KEY" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{ "scheduled_at": "2026-11-03T18:00:00-05:00", "scheduled_timezone": "America/New_York" }'
```

`scheduled_at` must carry an explicit UTC offset; it may be now or in the past, in which case sending starts as soon as audience compilation finishes (no minimum lead time). `scheduled_timezone` must be one of the six supported US IANA zones: `America/New_York`, `America/Chicago`, `America/Denver`, `America/Los_Angeles`, `America/Anchorage`, or `Pacific/Honolulu`. `daily_cap_bypass` is optional (default `false`): brands T-Mobile meters (Aegis-vetted, non-political) carry a per-brand daily T-Mobile limit and a project otherwise pauses at it each Pacific day and must be started again to continue; set it to `true` to run the whole project through in one pass, accepting that messages to T-Mobile recipients over the limit may fail and are still billed.

The Quickstart section of <https://politicalcomms.com/llms-full.txt> carries the same examples and surrounding context.

## Email (early access)

The `/v1/email` surface covers sending domains, sender identities, lists and their contacts, list imports, suppressions, campaigns, and templates: 40 operations in all.

**Every `/v1/email/*` endpoint returns `403 EMAIL_EARLY_ACCESS` until the email product reaches general availability.** Treat that response as expected, not as a bug, a bad key, or a permissions problem: do not retry it, and do not tell the operator their credentials are wrong. The contract is published and stable, so an integration can be written against it now and will work unchanged once the flag is lifted.

Facts that differ from the messaging surface, and that agents get wrong if they assume otherwise:

- **Email lists are keyset paginated.** They return `{ "data": [...], "has_more": bool, "next_cursor": string|null }` inside `data`, unlike the messaging endpoints, which return a plain array. Page until `next_cursor` is null. Cursors are opaque: never parse, construct, or reuse one across a different ordering.
- **There is no inbox and no inbound email.** We do not receive email, and there is no inbound email webhook. Replies go to the `reply_to` address on the sender identity. Do not build a polling loop looking for one.
- **There is no A/B testing** and no `email.opened` webhook event.
- **DNS is manual.** `POST /v1/email/domains` returns the records to publish; the platform never writes DNS and never asks for registrar credentials. Poll the domain until `status` is `active`.
- **Removing list contacts unsubscribes them, it does not delete them.** The rows carry the bounce and complaint history that stops a later re-import from resurrecting a suppressed address.
- **Check before scheduling.** `GET /v1/email/campaigns/{id}` returns a `blocked` array naming exactly what is stopping a schedule. Read it rather than calling schedule and interpreting the failure.
- **Do not retry a refused resume.** A campaign a deliverability breaker auto-paused twice returns `409 EMAIL_CAMPAIGN_RESUME_REQUIRES_SUPPORT`. There is no override on this surface; escalate to a human.
- **Test sends are real sends.** They are billed per recipient. They are excluded from campaign stats and never fire webhooks.
- **Lincoln drafts email templates, asynchronously and for money.** `POST /v1/email/templates/drafts` answers `202` with a draft id and the `unit_price` ($3.00 by default); poll `GET /v1/email/templates/drafts/{id}` until `status` is `ready` or `failed`, usually under two minutes. Do not treat the `202` as a finished draft, and do not busy-poll faster than about once every two seconds. The charge lands only when a draft becomes `ready`: a `failed` draft, whatever its `error_code` (`EMAIL_DRAFT_INVALID`, `EMAIL_DRAFT_MODEL_ERROR`, `INSUFFICIENT_BALANCE`), is never billed. Ask for a fresh draft rather than retrying a failed one. A wallet that cannot cover it returns `402 INSUFFICIENT_BALANCE` up front and creates no draft.
- **Images in a draft must already be email assets.** `image_media_ids` (at most six) must name media in the same organization with `usage` `email_asset`. Upload one with `POST /v1/media` and `usage: "email_asset"`, which is also the flag that puts an image in the email library rather than the texting one. `brand_id` is not accepted with an email asset, because email assets are organization-scoped.
- **Importing a list is one call over a URL you host.** `POST /v1/email/lists/import` fetches an HTTPS CSV (50 MB cap, SSRF-guarded), stages it, and commits, answering `202`; poll `GET /v1/email/lists/imports/{id}`. `mapping` is optional and common ESP exports are recognized; when no email column is found the call returns `400 VALIDATION_ERROR` with `details.headers` listing what was read, so send a mapping naming the right column rather than retrying the same body.
- **A validation export is polled, and `409` is the signal.** `POST /v1/email/lists/{id}/export` answers `202` with a `file_id`; `GET /v1/email/lists/{id}/export/{fileId}/download` answers `409 EXPORT_NOT_READY` until the worker has built the file, so treat that as "poll again" rather than an error, and do not re-queue. Send `{"state": "undeliverable"}` to narrow the CSV to one verdict class. Blank verdict columns mean the address has no cached verdict (never validated, or the 90-day cache expired), which is not the same as `false`.
- **Read `lint` on every template write.** `POST` and `PATCH` on `/v1/email/templates` return a `lint` object beside the template. A template with lint errors saves, but a campaign built on it will not schedule, so a caller that ignores `lint` discovers the problem at schedule time instead.

Five email webhook events subscribe alongside the message events: `email.delivered`, `email.bounced` (permanent bounces only), `email.complained`, `email.unsubscribed`, and `email.clicked` (unique, non-bot clicks). Each payload carries a stable `id` that is also the deduplication key.

## Rate limits and backoff

- **Limit:** per API key over a 60-second sliding window: 100 requests per minute for reads, 60 per minute for writes, 30 per minute for deletes.
- **Headers:** every response includes `X-RateLimit-*` headers communicating current usage and reset windows. Read them; do not count requests yourself.
- **Backoff:** when you approach the limit, pause until the reset window indicated by the headers. On a rate limit rejection, wait and retry after the reset rather than retrying immediately. Batch reads and cache list responses where the workflow allows it.

## Errors and retries

Errors return structured JSON:

```json
{
  "success": false,
  "error": "Human-readable message",
  "code": "machine_readable_code",
  "statusCode": 400
}
```

Retry guidance:

- **4xx (except rate limiting):** the request is wrong. Fix the request. Do not retry the same payload.
- **Rate limit rejections:** wait for the reset window from the `X-RateLimit-*` headers, then retry.
- **5xx:** transient. Retry with exponential backoff and an `Idempotency-Key` so the retry is safe.
- Always read `code` and `statusCode` from the body rather than parsing the error string.

## Idempotency

Write endpoints support the optional `Idempotency-Key` header. Set it on any request you might retry, and always on `POST /projects/{id}/schedule`. A UUID per logical operation is the correct pattern: retries with the same key will not double-schedule a send.

## Webhooks

Subscribe an HTTPS endpoint and the platform pushes events:

- `message.sent`
- `message.delivered`
- `message.failed`
- `message.replied`
- `link.clicked`

Email events (early access), which fire only once the email product is generally available:

- `email.delivered`
- `email.bounced` (permanent bounces only; soft bounces are deferred and never fire)
- `email.complained`
- `email.unsubscribed`
- `email.clicked` (unique, non-bot clicks)

**Signature verification is mandatory.** Each delivery carries an HMAC-SHA256 signature in the `X-Webhook-Signature` header, formatted `sha256=...`. Compute the HMAC-SHA256 of the raw request body with your webhook secret, compare against the header using a constant-time comparison, and reject anything that does not match before touching the payload.

**Delivery semantics:** respond with a 2xx quickly. Non-2xx responses trigger automatic retries, so design handlers to be idempotent on event delivery.

## MCP server for documentation lookup

Political Comms operates an MCP server, documented at <https://docs.politicalcomms.com/mcp>, with a discovery manifest at <https://politicalcomms.com/.well-known/mcp.json>. It exposes three tools:

- `search_political_comms` for searching Political Comms documentation and content
- `query_docs_filesystem_political_comms` for querying the documentation filesystem
- `submit_feedback` for sending feedback about the docs or platform

When you need current documentation during a build, use the MCP server rather than relying on training data.

## Machine-readable references

| Resource | URL |
|----------|-----|
| Index for agents | <https://politicalcomms.com/llms.txt> |
| Full reference (single fetch) | <https://politicalcomms.com/llms-full.txt> |
| Auth walkthrough | <https://politicalcomms.com/auth.md> |
| Pricing | <https://politicalcomms.com/pricing.md> |
| Site index | <https://politicalcomms.com/index.md> |
| OpenAPI 3.1 spec | <https://docs.politicalcomms.com/api-reference/openapi.json> |
| API docs | <https://docs.politicalcomms.com/api-reference/introduction> |
| MCP server | <https://docs.politicalcomms.com/mcp> |
| MCP discovery manifest | <https://politicalcomms.com/.well-known/mcp.json> |
