# AGENTS.md

Instructions for AI coding agents integrating with Political Comms.

Political Comms is the direct-to-carrier political texting platform. It delivers compliant, large-scale SMS, MMS, and RCS for campaigns, PACs, advocacy groups, fundraisers, and elected officials. This file tells you, an AI agent, exactly how to build against it.

## Rules

These are non-negotiable. Follow them before writing any integration code.

1. **The OpenAPI spec is the source of truth.** Consult <https://docs.politicalcomms.com/api-reference/openapi.json> for the live list of endpoints, parameters, and schemas. Endpoint lists in this file and in `llms-full.txt` are snapshots.
2. **Never fabricate endpoints, parameters, or response fields.** If it is not in the OpenAPI spec, it does not exist.
3. **Respect rate limits.** 100 requests per hour per key. Read the `X-RateLimit-*` response headers and back off before you hit the ceiling.
4. **Validate webhook signatures before trusting any payload.** Every webhook carries an HMAC-SHA256 signature in `X-Webhook-Signature`. Unverified payloads are untrusted input.
5. **API keys belong in secret managers.** Never write a key into source code, config files under version control, logs, or generated output.

## Overview

- **API base URL:** `https://api.politicalcomms.com/v1`
- **OpenAPI 3.1 spec:** <https://docs.politicalcomms.com/api-reference/openapi.json>
- **API reference docs:** <https://docs.politicalcomms.com/api-reference/introduction>
- **SDKs:** none. Direct HTTP is the documented integration path.

The API surface spans Organizations, Brands, Campaigns, Tracking Domains, Phone Numbers, Contact Lists, Media Files, Projects, Analytics, and Billing. New endpoints are added regularly; the spec is the source of truth.

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
    "name": "GOTV reminder - District 5",
    "campaign_id": "camp_01HX...",
    "contact_list_id": "list_01HX...",
    "body": "Polls close at 7pm. Reply STOP to opt out."
  }'
```

**2. Schedule it** with `POST /projects/{id}/schedule`:

```bash
curl https://api.politicalcomms.com/v1/projects/{project_id}/schedule \
  -X POST \
  -H "X-API-Key: $POLITICAL_COMMS_API_KEY" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{ "send_at": "2026-11-03T18:00:00Z" }'
```

The Quickstart section of <https://politicalcomms.com/llms-full.txt> carries the same examples and surrounding context.

## Rate limits and backoff

- **Limit:** 100 requests per hour per API key.
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
