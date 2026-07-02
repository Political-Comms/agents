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
- **SDKs:** none. Direct HTTP is the documented integration path.

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
    "name": "GOTV reminder - District 5",
    "campaign_id": "camp_01HX...",
    "contact_list_id": "list_01HX...",
    "body": "Polls close at 7pm. Reply STOP to opt out."
  }'
```

Schedule it:

```bash
curl https://api.politicalcomms.com/v1/projects/{project_id}/schedule \
  -X POST \
  -H "X-API-Key: $POLITICAL_COMMS_API_KEY" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{ "send_at": "2026-11-03T18:00:00Z" }'
```

Use an `Idempotency-Key` (a UUID per logical operation) on writes you might retry, always on schedule calls. Retries with the same key will not double-schedule.

## Rate limits

100 requests per hour per API key. Every response carries `X-RateLimit-*` headers with current usage and reset windows. Read the headers rather than counting requests. Back off before the ceiling; on a rate limit rejection, wait for the reset window before retrying.

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

## Webhooks

Events: `message.sent`, `message.delivered`, `message.failed`, `message.replied`, `link.clicked`.

Every delivery carries an HMAC-SHA256 signature in the `X-Webhook-Signature` header, formatted `sha256=...`. Compute HMAC-SHA256 over the raw request body with the webhook secret, compare with a constant-time comparison, and reject mismatches before reading the payload.

Respond 2xx quickly. Non-2xx triggers automatic retries, so make handlers idempotent on event delivery.

## References

- Full reference in one fetch: <https://politicalcomms.com/llms-full.txt>
- Index: <https://politicalcomms.com/llms.txt>
- Auth: <https://politicalcomms.com/auth.md>
