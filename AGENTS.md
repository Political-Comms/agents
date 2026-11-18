# AGENTS.md

Instructions for AI coding agents integrating with Political Comms.

Political Comms is the direct-to-carrier political texting platform. It delivers compliant, large-scale SMS, MMS, and RCS for campaigns, PACs, advocacy groups, fundraisers, and elected officials. This file tells you, an AI agent, exactly how to build against it.

## Rules

These are non-negotiable. Follow them before writing any integration code.

1. **The OpenAPI spec is the source of truth.** Consult <https://docs.politicalcomms.com/api-reference/openapi.json> for the live list of endpoints, parameters, and schemas. Endpoint lists in this file and in `llms-full.txt` are snapshots.
2. **Never fabricate endpoints, parameters, or response fields.** If it is not in the OpenAPI spec, it does not exist.
3. **Respect rate limits.** 600 requests per minute per key, the same for every scope: bursts of up to 600 at once, refilling at 10 per second. Read the `X-RateLimit-*` response headers and back off before you hit the ceiling.
4. **Validate webhook signatures before trusting any payload.** Every webhook carries an HMAC-SHA256 signature in `X-Webhook-Signature`. Unverified payloads are untrusted input.
5. **API keys belong in secret managers.** Never write a key into source code, config files under version control, logs, or generated output.

## Overview

- **API base URL:** `https://api.politicalcomms.com/v1`
- **OpenAPI 3.1 spec:** <https://docs.politicalcomms.com/api-reference/openapi.json>
- **API reference docs:** <https://docs.politicalcomms.com/api-reference/introduction>
- **SDKs:** official TypeScript client (`npm install @political-comms/sdk`), Python client (`pip install political-comms`), CLI (`npx @political-comms/cli`), and MCP server (`npx -y @political-comms/mcp`). Direct HTTP against the spec also works.

The API surface spans Organizations, Brands, Campaigns, Tracking Domains, Webhook Endpoints, Phone Numbers, Contact Lists, Tags, Media Files, Projects (including triggered projects), Conversations, Analytics, Billing, and Email. `GET /phone-numbers` includes numbers a parent organization shared with the key's organizations (`shared: true`, `nickname`; the owner's names are null on those rows). Brand, campaign and owned tracking-domain rows carry a `notes` field: the account's internal note (up to 100 characters) or null; it is never shown to recipients. New endpoints are added regularly; the spec is the source of truth.

## Operation snapshot (OpenAPI 1.14.0, 2026-11-17)

Every /v1 operation at the snapshot date; the spec is the live list.

### Projects

- GET /v1/projects: List Projects
- POST /v1/projects: Create Project
- GET /v1/projects/stats: Get Project Stats (all)
- GET /v1/projects/{id}: Get Project
- PATCH /v1/projects/{id}: Update Project
- POST /v1/projects/{id}/copy: Copy Project
- POST /v1/projects/{id}/end: End Project
- POST /v1/projects/{id}/hook: Create Hook URL
- POST /v1/projects/{id}/pause: Pause Project
- POST /v1/projects/{id}/recipients: Add Recipient
- POST /v1/projects/{id}/schedule: Schedule Project
- POST /v1/projects/{id}/start: Start Project (a tag trigger's first start needs `audience`)
- GET /v1/projects/{id}/stats: Get Project Stats (single)
- GET /v1/projects/{id}/survey-results: Get Survey Results
- POST /v1/projects/{id}/test: Test Project
- GET /v1/projects/{id}/throughput: Get Project Throughput
- POST /v1/projects/{id}/unschedule: Unschedule Project

### Conversations

- GET /v1/conversations: List Conversations
- GET /v1/conversations/{conversation_id}: Get Conversation
- GET /v1/conversations/{conversation_id}/messages: List Conversation Messages
- POST /v1/conversations/{conversation_id}/messages: Reply to Conversation

### Phone Numbers

- GET /v1/phone-numbers: List Phone Numbers

### Toll-Free Verifications

- GET /v1/toll-free-verifications: List Toll-Free Verifications
- GET /v1/toll-free-verifications/{id}: Get Toll-Free Verification

### Brands

- GET /v1/brands: List Brands

### 10DLC Campaigns

- GET /v1/campaigns: List Campaigns
- GET /v1/campaigns/{id}/throughput: Get Campaign Throughput

### Email Campaigns

- GET /v1/email/campaigns: List Campaigns
- POST /v1/email/campaigns: Create Campaign
- GET /v1/email/campaigns/{id}: Get Campaign
- POST /v1/email/campaigns/{id}/end: End Campaign
- POST /v1/email/campaigns/{id}/hook: Create Hook URL
- POST /v1/email/campaigns/{id}/pause: Pause Campaign
- POST /v1/email/campaigns/{id}/recipients: Add Recipient
- POST /v1/email/campaigns/{id}/resume: Resume Campaign
- POST /v1/email/campaigns/{id}/schedule: Schedule Campaign
- GET /v1/email/campaigns/{id}/stats: Get Campaign Stats
- POST /v1/email/campaigns/{id}/unschedule: Unschedule Campaign

### Email Templates

- GET /v1/email/templates: List Email Templates
- POST /v1/email/templates: Create Email Template
- GET /v1/email/templates/{id}: Get Email Template

### Email Senders

- GET /v1/email/senders: List Sender Identities
- GET /v1/email/senders/{id}: Get Sender Identity

### Email Suppressions

- GET /v1/email/suppressions: List Suppressions
- POST /v1/email/suppressions: Add Suppressions
- DELETE /v1/email/suppressions: Remove Suppressions (needs `public:delete`)

### Contact Lists

- GET /v1/contact-lists: List Contact Lists
- POST /v1/contact-lists/import: Import List
- GET /v1/contact-lists/{id}: Get Contact List
- DELETE /v1/contact-lists/{id}: Delete Contact List (needs `public:delete`)
- POST /v1/contact-lists/{id}/analyze: Analyze Contact List
- GET /v1/contact-lists/{id}/download: Download Contact List CSV

### Tags

- GET /v1/tags: List Tags (rows carry `live_triggers`)
- POST /v1/tags/contacts: Add Contacts to Tag by Name (creates the tag on first use; returns `created`; a `contact_list_id` source returns `202` queued; `fire_triggers`, see Tags)
- DELETE /v1/tags/contacts: Remove Contacts from Tag by Name (needs `public:delete`; never creates)
- POST /v1/tags: Create Tag
- PATCH /v1/tags/{id}: Rename Tag
- DELETE /v1/tags/{id}: Delete Tag (needs `public:delete`)
- POST /v1/tags/{id}/contacts: Add Contacts to Tag (`fire_triggers` chooses whether live tag triggers message the newly tagged; see Tags)
- DELETE /v1/tags/{id}/contacts: Remove Contacts from Tag (needs `public:delete`)

### Media Files

- GET /v1/media: List Media Files
- POST /v1/media: Import Media
- GET /v1/media/{id}: Get Media File
- DELETE /v1/media/{id}: Delete Media File (needs `public:delete`)

### Sending Domains

- GET /v1/email/domains: List Sending Domains
- GET /v1/email/domains/{id}: Get Sending Domain

### Tracking Domains

- GET /v1/tracking-domains: List Tracking Domains

### Webhook Endpoints

- GET /v1/webhook-endpoints: List Webhook Endpoints
- GET /v1/webhook-endpoints/{id}: Get Webhook Endpoint

### Organizations

- GET /v1/hierarchy: Get Hierarchy
- GET /v1/organizations: List Organizations

### Billing

- GET /v1/ledger/usage: Get Usage
- GET /v1/ledger/usage/by-initiator: Get Usage by Initiator

### Analytics

- GET /v1/messages/stats: Get Message Stats

## Authentication

Every request carries an API key in the `X-API-Key` header:

```
X-API-Key: pc_live_1234567890abcdef
```

Key facts:

- Keys carry up to three permissions: `public:read` (GET, HEAD), `public:write` (POST, PATCH, PUT), and `public:delete` (DELETE, which also needs `public:write`: `DELETE /contact-lists/{id}`, `DELETE /media/{id}`, `DELETE /email/suppressions`, `DELETE /tags/{id}`, `DELETE /tags/{id}/contacts`, `DELETE /tags/contacts`). New keys get read and write; delete is opt-in when the human creates the key and requires write. Permissions cannot be edited; ask the operator for a new key. Each request needs only the permission for its method: GET and HEAD need `public:read`, POST, PATCH and PUT need `public:write`, DELETE needs `public:write` and `public:delete`. A write does not need `public:read`: a write-only key can create and change things but cannot read, so do not expect to GET with it. A key missing the permission gets 403 "Insufficient permissions".
- Keys are prefixed `pc_live_`.
- Key provisioning is human-initiated. A person generates keys in the dashboard at <https://app.politicalcomms.com/> under Admin → API. An agent cannot self-provision a key; ask the operator to create one.
- Keys are shown once at creation. Store them in a secret manager immediately.
- Read the key from an environment variable or secret manager at runtime. Never hardcode it.

The full agent-oriented auth walkthrough lives at <https://politicalcomms.com/auth.md>.

## Core workflow: create, then schedule

A "project" is the unit of work: a composed message body, optional media, link tracking, and the contact list it targets. Sending is two calls, with an optional test in between: `POST /projects/{id}/test` accepts `test_contacts`, and each entry may carry an optional `merge_values` object (tag name to value) so it renders without sampling a list contact.

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

Notes on the body: `brand_id` and `campaign_id` are required on the default `10dlc` channel and omitted for `toll-free`; `contact_list_ids` is optional: omit it to create a draft and attach lists or tags later via `PATCH /projects/{id}`; a project is complete with at least one list or one `tag_ids` entry (see Tags below). Validation is strict: unknown properties are rejected with a 400.

Test phone numbers: To rehearse a specific outcome without texting a real phone, send the test to a test phone number: any US number ending 555-0100 through 555-0199 (any area code). It never reaches a carrier, and the last two digits fix the result: 00 delivered; 01 failed 003 (blocked as spam); 02 failed 004 (rejected by destination); 03 failed 021 (not a mobile phone); 04 failed 012 (invalid number); 05 failed 300 (recipient opted out); 06 failed 005 (expired); 10 delivered, then texts back a reply; 11 delivered, then texts back STOP (added to the sender's opt-out list); every other ending delivered. Webhooks fire as normal with carrier `Test carrier`, and the message is billed at the normal rate. See https://docs.politicalcomms.com/api-reference/testing.

Editing: `PATCH /projects/{id}` works on broadcast projects in `draft`, `awaiting_test`, `ready`, or `paused` (other statuses return `409 INVALID_STATE_TRANSITION` with `details.allowed`); only `organization_id` and `type` are immutable (`400 FIELD_NOT_EDITABLE`). `channel` (`10dlc`, `toll-free`, `short-code`), `brand_id`, `campaign_id`, and `toll_free_verification_id` are editable, but sending any of them requires the full setup for the effective channel (10dlc: `brand_id` and `campaign_id`; toll-free: `toll_free_verification_id`; short-code: none of the three) plus `phone_number_ids`, with the same checks and error codes as create; `409 CAMPAIGN_SUSPENDED` for a suspended campaign, `409 CAMPAIGN_EXPIRED` for an expired one, and `400 ASSET_OUT_OF_SCOPE` when the project's lists or media belong to the old brand or registration (send new `contact_list_ids`, `suppression_list_ids`, and `media_ids` in the same request). Any edit moves the project to `awaiting_test`. To resume a paused project after an edit: `PATCH`, then `POST /projects/{id}/test`, then `POST /projects/{id}/schedule`; recipients already sent are never re-sent. `POST /projects/{id}/schedule` returns `409 TOLL_FREE_NOT_VERIFIED` (`details.phoneNumberIds`) when a toll-free number has not finished verification.

**2. Schedule it** with `POST /projects/{id}/schedule`:

```bash
curl https://api.politicalcomms.com/v1/projects/{project_id}/schedule \
  -X POST \
  -H "X-API-Key: $POLITICAL_COMMS_API_KEY" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{ "scheduled_at": "2026-11-03T18:00:00-05:00", "scheduled_timezone": "America/New_York" }'
```

`scheduled_at` must carry an explicit UTC offset; it may be now or in the past, in which case sending starts as soon as audience compilation finishes (no minimum lead time). `scheduled_timezone` must be one of the six supported US IANA zones: `America/New_York`, `America/Chicago`, `America/Denver`, `America/Los_Angeles`, `America/Anchorage`, or `Pacific/Honolulu`. `daily_cap_bypass` is optional (default `false`): for brands with a T-Mobile daily limit on file (political brands have none today), T-Mobile recipients over the limit otherwise send at a reduced rate; set it to `true` to skip the reduced rate and send them at full speed, accepting that a carrier rejection still pauses the project and rejected messages are still billed. It holds only until the next schedule or resume call rewrites it, so send it on every schedule or resume call. Scheduling also resumes a paused project.

Carrier throughput limits: T-Mobile caps message parts (160-character SMS segments) per day per brand for brands that have a T-Mobile daily limit on file (political brands have none today), resetting at midnight Pacific. The cap is a speed limit, not a stop. Past it, T-Mobile and unknown-carrier recipients keep sending at about 300 message parts per minute per campaign while the platform watches receipts; after about 300 over-cap receipts with under 5% failures the rate rises to about 3,000 parts per minute. Other carriers are never slowed. Only the carrier pauses a project: if T-Mobile rejects messages over the cap (error code 016, or 20 or more failures that are at least half of the newest three minutes of T-Mobile receipts), the campaign's T-Mobile lane closes for the Pacific day and, once only T-Mobile and unknown-carrier recipients remain, the project auto-pauses (`GET /projects/{id}` returns `status: paused`, `pause_reason: brand_daily_cap`, `auto_paused: true`). Rejected messages are still billed. Run `POST /contact-lists/{id}/analyze` first and only T-Mobile recipients are affected. Resume with `POST /projects/{id}/schedule` (it accepts paused projects): schedule it for the next day during sending hours (the project's sending window: 8 AM to 10 PM in the time zone of most recipients, or 8 AM Eastern to 10 PM Pacific when no zone holds a majority) with a morning `scheduled_at` to wait for the reset, or call it without one to reopen the lane and try T-Mobile again at the reduced rate (it pauses again if T-Mobile is still rejecting). `daily_cap_bypass: true` skips the reduced rate and sends T-Mobile at full speed past the cap; a carrier rejection still pauses the project. Never resume at midnight. Every 10DLC campaign with an AT&T rate on file, political brands included, has its AT&T traffic delivered at that rate, counted in message parts (separate SMS and MMS rates; a two-part text counts twice). The platform sends AT&T recipients right away and the carrier queue delivers them at that rate, usually within about an hour; only in the last 90 minutes of the recipients' sending window does the platform hold back AT&T recipients that could not be delivered before the window closes. A campaign with no rate on file has no AT&T rate applied. AT&T never slows other carriers and never pauses a project. Check before scheduling with `GET /campaigns/{id}/throughput` (`t_mobile.daily_cap`, `used_today`, `remaining_today`, `att.sms_tpm`, `att.mms_tpm`). The lanes are independent: `t_mobile` is null when the brand has no T-Mobile daily limit on file or it has not synced yet, `att` is null only when the campaign has no AT&T rate on file (`sms_tpm` and `mms_tpm` can each be null), and `carrier_metered` is true if either lane applies (a political brand with an AT&T rate returns true with `t_mobile` null and `att` populated; false with both null when neither applies). `GET /projects/{id}/throughput` gives `t_mobile.estimated_recipients`, `att.estimated_minutes` (minutes AT&T needs at `att.tpm` for what the campaign already has waiting plus this project's AT&T recipients, accounting for the parts in the project's text), `carrier_coverage`; `t_mobile.will_pause` is always `false` and `t_mobile.estimated_send_days` is always `1` (kept for compatibility), results are cached up to 60 seconds, and a timed-out estimate returns 503 `CARRIER_ESTIMATE_TIMEOUT` (retry later). A null means not known right now, never zero. Poll for `paused`, not only `completed`: a `brand_daily_cap` pause means T-Mobile rejected messages over the cap, not that the project is stuck. Known `pause_reason` values: `brand_daily_cap` (T-Mobile rejected messages over the cap; schedule it for the next day's sending hours with a morning `scheduled_at`, or call schedule without one to try T-Mobile again at the reduced rate), `quiet_hours` (paused when the project's sending window closes: 10 PM in the time zone of most recipients, or 10 PM Pacific when no zone holds a majority; restart manually once the window opens), `carrier_block_rate`, `unregistered_campaign`, `insufficient_funds_auto_recharge_failed`, `insufficient_funds_ancestor`, `shared_phone_revoked`, `phone_released`, `organization_deleted`, `manual` or a short free-text reason (up to 100 characters) set by a user; other values may appear. Every auto-pause needs a manual restart via `POST /projects/{id}/schedule`. You can pause a project yourself with `POST /projects/{id}/pause` (from `active`, `sending`, or `scheduled`; optional `reason` up to 100 characters becomes `pause_reason`; already paused returns 200; other statuses return `409 INVALID_STATE_TRANSITION` with `details.allowed`), and resume it the same way, with `POST /projects/{id}/schedule`.

Contact lists: `POST /contact-lists/{id}/analyze` runs line-type analysis and is billable per lookup, charged to the list's organization. Send an `Idempotency-Key`; `402 INSUFFICIENT_BALANCE` queues nothing; `202` returns `cost_cents` and `numbers_queued`; `200` with `analysis.status` `complete` (cost 0) means nothing was left to analyze; a run already in progress returns `202` with cost 0. It is async: poll `GET /contact-lists/{id}` until `analysis.status` is `complete`, or subscribe to the opt-in `contact_list.analyzed` webhook (payload `id`, `list_id`, `name`, `analysis`, `analyzed_url`). `GET /contact-lists/{id}` returns `downloads.original_url` and `downloads.analyzed_url` (null until complete); `GET /contact-lists/{id}/download?type=original|analyzed` streams the CSV (`type` required; analyzed adds Phone Type, Carrier Name, Is Mobile, Is Opted Out, City, State; `409 ANALYSIS_NOT_COMPLETE` before completion; `404` outside the key's scope). The download URLs need your `X-API-Key` and are not time-limited (the analyzed file is kept for one year). `POST /projects`, project update, and `POST /projects/{id}/schedule` return `409 LIST_ANALYSIS_IN_PROGRESS` while a contact list on the project is being analyzed; `details.lists` names each list. The request changes nothing: wait for `analysis.status` `complete` (or `contact_list.analyzed`), then retry it. A contact list of your own whose contacts were removed 12 months after upload returns `409 LIST_ARCHIVED` on the same three calls (`details.lists` is `{ id, name }`; choose another list or upload the file again), and schedule returns `409 NO_SENDABLE_CONTACTS` when the project's lists hold no contact that can receive a text.

The Quickstart section of <https://politicalcomms.com/llms-full.txt> carries the same examples and surrounding context.

**Account setup can block creates.** `POST /projects`, `POST /projects/{id}/copy`, and `POST /v1/email/campaigns` (and their dashboard equivalents: new brands, campaigns, toll-free verifications, phone number purchases, email domains and senders) return `403 ONBOARDING_INCOMPLETE` once an organization's 14-day account setup grace window has passed with its business profile or funding step still incomplete. `details.missingSteps` names what's outstanding (`profile`, `funding`, or both) and `details.onboardingUrl` is `/onboarding`. This never affects anything already running: existing sends, schedules, and conversations are unaffected. Escalate to a human operator; an agent cannot complete account setup on the organization's behalf.

**Some features require enablement by Political Comms.** Every write (`POST`/`PUT`/`PATCH`/`DELETE`) under `/v1/email/*` returns `403 ENTITLEMENT_REQUIRED` if the organization is not entitled to that feature. `details.entitlement` names the missing key (`tendlc`, `toll_free`, `short_code`, `data_purchases`, `email`, `stripe_connect`, `phone_number_sharing`, or `sub_orgs`). Escalate to a human operator to contact support; an agent cannot self-enable an entitlement.

**Sending can be paused.** `POST /projects`, `POST /projects/{id}/schedule`, `POST /v1/conversations/{id}/messages`, and `POST /v1/email/campaigns/{id}/schedule` return `409 SENDING_PAUSED` if Political Comms has paused sending for the organization or platform-wide. `details.scope` is `organization` or `platform`. Do not retry; escalate to a human operator.

## Tags

A tag labels a contact (a phone number or an email address) inside one organization. `/v1/tags` has eight operations: `GET /v1/tags` (sorted by name, not paginated; optional `organization_id`), `POST /v1/tags`, `PATCH /v1/tags/{id}`, `DELETE /v1/tags/{id}`, `POST /v1/tags/contacts` and `DELETE /v1/tags/contacts` (by name), and `POST /v1/tags/{id}/contacts` and `DELETE /v1/tags/{id}/contacts` (by id). Reads need `public:read`, writes `public:write`, and the three DELETEs also `public:delete`. A tag object is `{ id, name, kind, activity_event, within_days, member_count, member_count_capped, created_at, organization_id }`.

- **Two kinds.** A `manual` tag has members you add and remove: create it with `{ "name": "Door knockers" }` (1 to 64 characters, unique among the organization's manual tags ignoring case, else `409 TAG_NAME_TAKEN`). An `activity` tag has no members and applies by itself while true: create it with `{ "activity_event": "donated" | "clicked" | "responded" | "sent", "within_days": 1..365 | null }`. Creating one that already exists returns `200` with it instead of `201`. Activity tags cannot be renamed, added to, or removed from (`400 TAG_NOT_MANUAL`); `member_count` is null on them. A manual tag's `member_count` stops at 10,000 (`member_count_capped`).
- **Tag by name (preferred).** `POST /v1/tags/contacts` takes `{ "name": "Door knockers", "identifiers": [..] }` or `{ "name": ..., "contact_list_id": "<id>" }` (plus optional `organization_id`) and tags in one call: if the organization has no tag with that name, it is created, so there is no create step. Names are trimmed, 1 to 64 characters, and not case sensitive (the first spelling stored is kept). List tags first with `GET /v1/tags` and reuse an existing name rather than inventing a near-duplicate. With `identifiers` the answer is `{ tag, created, added, invalid }`: check `created`, because `true` when you expected an existing tag means a misspelled name made a new one. With `contact_list_id` the work is queued in the background and the call returns `202` with `{ tag, created, queued: true, job_id }` and no counts: do not wait for counts, and do not poll `job_id` (there is no job endpoint); check the tag's `member_count` on `GET /v1/tags` later (the `member_count` in the 202 is the count before the job runs). The tag is still created at once. `DELETE /v1/tags/contacts` (same body, needs `public:delete`) answers `{ tag, removed, not_tagged, invalid }` for identifiers (`202` with `{ tag, queued: true, job_id }` for a list), never creates a tag, and returns `404 NOT_FOUND` when no tag has the name. A name always means a manual tag, even one matching an activity tag's generated name (`Clicked` makes a manual tag beside the activity tag); `404 LIST_NOT_FOUND` creates nothing. Sends still take tag ids, not names: use `tag.id` from the answer or `GET /v1/tags`.
- **Add or remove contacts by id.** `POST` and `DELETE /v1/tags/{id}/contacts` take exactly one of `{ "identifiers": [..1 to 10000 US phone numbers or email addresses] }` or `{ "contact_list_id": "<id>" }`. Identifiers answer `{ added, invalid }` and `{ removed, not_tagged, invalid }`; a `contact_list_id` answers `202` with `{ queued: true, job_id }` and no counts (check `member_count` on `GET /v1/tags` later). An `Idempotency-Key` replay returns the stored `202` and queues nothing new. `409 TAG_BUSY` means an import is still writing the same contacts: retry in about a minute. `404 LIST_NOT_FOUND` means `contact_list_id` is not a list of the tag's organization.
- **Delete.** `DELETE /v1/tags/{id}` returns `409 TAG_IN_USE` with `details.projects` and `details.emailCampaigns` (up to 10 each, `{ id, name, status }`) while a scheduled, compiling, active, sending, or paused send uses the tag, and a live triggered project or campaign that listens for the tag blocks it too (`details.triggeredItems`, `{ id, name }`). Do not retry it; remove the tag from those sends, end the triggered item, or wait.
- **Send to tags.** `POST /v1/projects` and `PATCH /v1/projects/{id}` (broadcast and survey projects) and `POST /v1/email/campaigns` take `tag_ids`, `require_tag_ids`, and `suppression_tag_ids` (uuid arrays, up to 50 each). `tag_ids` means ANY: send to everyone carrying at least one, in union with the lists. `require_tag_ids` means ALL: keep only contacts carrying every one of them. `suppression_tag_ids` leaves out contacts carrying any of them for this send only. A tag cannot be in both `tag_ids` and `suppression_tag_ids`, or both `require_tag_ids` and `suppression_tag_ids` (`400`), and every id must be a tag of the send's organization. Triggered projects and campaigns refuse tags as recipients (a tag can be their `trigger`); survey projects take them.
- **A project or campaign needs at least one list or one `tag_ids` entry.** `contact_list_ids` on a project or survey PATCH may be `[]` when a tag remains the source; an edit that leaves a survey with neither a list nor a source tag is `400` unless it is a draft, and a survey created with neither is a draft. `list_ids` on `POST /v1/email/campaigns` is no longer required: a non-triggered campaign needs `list_ids` or a non-empty `tag_ids`. Editing tags on a project is a content edit: it returns to `awaiting_test`. Project and email campaign objects return the three arrays.
- **Suppression tags are not opt-outs.** A suppression tag is a per-send choice. Opt-outs and unsubscribes always apply and are never changed by tagging.
- **A tagged contact needs a list row to be sendable.** Tagging does not record consent. A tagged phone is texted only if it is on a live list the project's sender can use (unscoped, or scoped to the project's own brand, toll-free number, or short code); a tagged address is emailed only if it is subscribed on a list the sending domain can use. A tagged contact on no such list is skipped, so add them to a list rather than retrying.
- **Tag at import.** `POST /v1/contact-lists/import` with `purpose: "audience"` takes `tag_ids` (manual tags, up to 20), `tag_names` (up to 20, created if missing), and `tags_column` (a header of the file; each cell holds tag names separated by commas, semicolons, or pipes; a column that would create more than 100 new tags makes the import `failed` on `GET /v1/contact-lists/{id}`, with `import.error.code` `TAG_COLUMN_TOO_MANY_TAGS`). A `do_not_contact` import takes no tags (`400`).
- **Tag triggers send messages: check before you tag.** A manual tag can be the trigger of a live triggered project or email campaign, and then tagging a contact into it MESSAGES that person (and charges the account). `GET /v1/tags` rows carry `live_triggers` (`[{ id, name, channel }]`, empty when none). Before any call that adds contacts to a tag, read `live_triggers` for that tag. If it is not empty, tell the user which projects or campaigns would start and that the people tagged will be messaged, and proceed only as they confirm. `fire_triggers` (boolean, on `POST /v1/tags/contacts`, `POST /v1/tags/{id}/contacts` and `POST /v1/contact-lists/import`) chooses whether the newly tagged people are sent to: it defaults to `true` with `identifiers`, and to `false` with a `contact_list_id` and on an import. Never send `fire_triggers: true` for a `contact_list_id` or an import without the user's explicit confirmation of that send, and send `fire_triggers: false` with `identifiers` whenever the user only wants the tag applied. Report `triggers.entered` (people entered, present when the tag has live triggers) from a `200`; a queued list or import echoes `fire_triggers` in its `202`. Re-tagging a contact who already has the tag never fires. The `DELETE` calls take no `fire_triggers` (`400`).
- **Scope.** A tag in another organization your key cannot reach is `404 NOT_FOUND`, the same as a missing one; `organization_id` outside your hierarchy is `403 ORG_ACCESS_DENIED`. Every write takes an optional `Idempotency-Key`.

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

A **triggered** text project or email campaign has no contact list. Each person is added when its `trigger` fires and gets one message, right away, after a delay, or the next day at a set time. It stays active until paused or ended. The type is fixed at creation.

`trigger` has the same shape for text and email (snake_case): `source` is `api`, `donation`, `event`, or `tag`; `start` is `immediate` (default) or `next_day`; `delay_minutes` is 0 to 43200 and must be 0 with `next_day`. A person is sent a triggered project or campaign once: `reentry` was removed (`{ "mode": "once" }` is still accepted and ignored so a trigger read back can be sent as it is; any other value is `400`; never send `reentry`). `next_day` sends the calendar day after the trigger fired at `sequence_send_time` (`HH:MM`) in `scheduled_timezone` (text) or `timezone` (email, default America/New_York), one of America/New_York, America/Chicago, America/Denver, America/Los_Angeles, America/Anchorage or Pacific/Honolulu; both are required in the same write with `next_day` (`400` names the missing field). A text's quiet hours (8 AM to 10 PM local) can only move a send later.

- `donation`: `donation.integrations` (1 to 20 of `{ provider: "winred" | "anedot", id }`), `donor` (`any`, `first_time`, `returning`), `gift` (`any`, `one_time`, `recurring`), `amount_min` / `amount_max` in dollars, `page_names`, `source_codes`.
- `event`: `event.event` is `text_link_click`, `text_reply`, `email_open`, `email_click`, or `email_reply`; `event.sources` is `"any"` (never includes another triggered item) or `{ "ids": [...] }` (project ids for `text_*`, campaign ids for `email_*`); `link_url` optional on `email_click`. Email opens are approximate.
- `api`: you or an inbound hook add recipients.
- `tag`: `tag.tag_ids` (1 to 20 distinct ids of the organization's own manual tags; `400` at `trigger.tag.tag_ids.<index>` otherwise). A tagged phone triggers a text project and a tagged email address an email campaign. Re-tagging a contact who already has the tag never fires, and untag then re-tag is still once. Once started, a tag trigger cannot be pointed at other tags (`400` on `trigger`): end it and create a new one. A tagged phone on a list the project can use is texted with that row's merge fields; one on no list is texted with merge-field defaults; one whose only list rows belong to another brand, toll-free number or short code is not texted and has not used its one entry. Opt-outs always apply.

Workflow:

1. **Create with a trigger.** Text: `POST /v1/projects` with `"type": "triggered"`, the broadcast body without `contact_list_ids` or `suppression_list_ids`, and a required `trigger`. Email: `POST /v1/email/campaigns` with a `trigger` instead of `list_ids` (`list_ids` or `tag_ids` is required unless `trigger` is set; a trigger cannot be combined with `list_ids` or tags, `400`; a `sequence` is accepted).
2. **Test it.** `POST /v1/projects/{id}/test` for text; the email test send for a campaign. To rehearse a specific outcome without texting a real phone, send the test to a test phone number: any US number ending 555-0100 through 555-0199 (any area code). It never reaches a carrier, and the last two digits fix the result: 00 delivered; 01 failed 003 (blocked as spam); 02 failed 004 (rejected by destination); 03 failed 021 (not a mobile phone); 04 failed 012 (invalid number); 05 failed 300 (recipient opted out); 06 failed 005 (expired); 10 delivered, then texts back a reply; 11 delivered, then texts back STOP (added to the sender's opt-out list); every other ending delivered. Webhooks fire as normal with carrier `Test carrier`, and the message is billed at the normal rate. See https://docs.politicalcomms.com/api-reference/testing.
3. **Start it.** Text: `POST /v1/projects/{id}/start` (also resumes a paused project; `409 CONFLICT` for a project that is not triggered) takes an optional body `{ "audience": "current_and_new" | "new_only" }`, required for a tag trigger's first start and refused (`400` at `audience`) for any other project: `current_and_new` also enters everyone who carries the tag now, in the background, which MESSAGES all of them, so get the user's explicit confirmation; `new_only` enters only contacts tagged from then on. Email: `POST /v1/email/campaigns/{id}/schedule` with no `scheduled_at` and the same `audience` rule. Pause a text project with `POST /v1/projects/{id}/pause`.
4. **Add recipients, or hand out the hook URL.** `POST /v1/projects/{id}/recipients` with `{ "phone_number": "+12025550141", "fields": { "first_name": "Ada" }, "occurrence_id": "evt_123" }`, or `POST /v1/email/campaigns/{id}/recipients` with `email`. Send an `occurrence_id` so a retry is a no-op. Answers: `202 { status: "queued", send_at }`, `200 { status: "already_entered", reason }` (`duplicate`, `already_entered`), `409 CONTACT_OPTED_OUT`, `409 CONFLICT` (not triggered, or not active: a paused or never started item refuses recipients rather than queueing them), `422 VALIDATION_ERROR` with `details { field, reason }`, `404`. `POST /v1/projects/{id}/hook` and `POST /v1/email/campaigns/{id}/hook` return `201 { hook_url, endpoint_url, secret }` once; a new call retires the old ones at once. `hook_url` carries the secret in its path; when the caller can set headers, post to `endpoint_url` with the `secret` in `X-Hook-Secret` instead, which keeps the secret out of URLs. `POST {hook_url}` takes a JSON or form body (`phone_number` or `phone`, or `email`; optional `occurrence_id`; other top-level scalar keys become fields), needs no API key, answers like the recipients endpoints, and rate limits at 600 requests a minute per hook. It works with the generic webhook step in Zapier, Make, and n8n; there is no published Zapier app. Treat the URL as a secret.
5. **End it.** `POST /v1/projects/{id}/end` or `POST /v1/email/campaigns/{id}/end` (no body). Ending is final and people still waiting are not sent; pausing is reversible.

Rules to respect: a text that comes due between 10 PM and 8 AM in the recipient's time zone (by area code) is held until 8 AM, email has no hold; opted-out numbers and suppressed addresses are never sent to; fields passed with a recipient are merge tags (`{first_name}`), are stored on the recipient, and appear in reports, exports, and outbound webhook `contact.custom_fields`; donation triggers carry `first_name`, `last_name`, `email`, `address`, `address_2`, `city`, `state`, `zip`, `employer`, `occupation`, `amount`, `page_name`, and `source_code` as merge tags. `GET /v1/projects/stats` rows for triggered projects add `waiting_recipients` and `skipped_recipients`. 

### Follow-ups

A broadcast or triggered **text** project can send follow-ups after its first message: `follow_ups` on `POST /v1/projects` and `PATCH /v1/projects/{id}` (`null` clears), returned on `GET /v1/projects/{id}` (`null` when none; steps then carry `media_urls`). Shape: `{ "steps": [{ "message_text": "...", "media_ids": [..up to 10, MMS projects only], "wait": { "unit": "days" | "hours", "amount": n } }], "stop_on_reply": bool, "stop_on_click": bool }`. 1 to 9 steps; days 1 to 365, hours 1 to 72; at least one stop flag must be true (`400`); an opt-out always ends a person's follow-ups; a person whose first message failed to deliver gets none; a person is sent each step at most once; each follow-up is charged when it sends. A day wait sends N days after the previous message at `sequence_send_time` in `scheduled_timezone` (send both in the same write when any step waits days); an hour wait sends N hours after the previous message; a recipient's quiet hours can only move a step later. A survey cannot have follow-ups (`400`). A broadcast with follow-ups is scheduled (a time, or now), in the same time zone as its follow-ups when they wait days. The steps are locked once the project has started (`409`: end it and copy it to change them); `scheduled_timezone` and `sequence_send_time` can still change, and people waiting keep their place. `GET /v1/projects/stats` rows add `exited_recipients` (`replied`, `clicked`, `opted_out` counts). Email follow-ups are the `sequence` object on `POST /v1/email/campaigns` (steps of `template_id`, `delay_days`, `send_time`; `stop_on_reply` default true, `stop_on_click` default false, at least one true): a campaign with follow-ups reads `type: "broadcast"` (`type` is `broadcast` or `triggered`; there is no `sequence` type), and beside a `trigger` follow-ups run on each person's own clock: `delay_days` must be 1 to 365 (0 is `400`) and counts from the calendar day that person's previous email was sent, in the campaign's `timezone`; email follow-ups wait whole days only.

## Email

The `/v1/email` surface covers sending domains and sender identities (read-only), suppressions (unsubscribes, bounces, complaints), campaigns, and templates: 21 operations in all. Lists are one family for phones and emails under `/v1/contact-lists`: created, with their phones and/or emails, only by `POST /v1/contact-lists/import`, and deleted with `DELETE /v1/contact-lists/{id}` (which removes the list's emails too). Setup, curation, and paid workflows are dashboard features rather than API endpoints: registering domains and senders, renaming lists, result exports, campaign test sends, and Lincoln drafting. A triggered campaign is the exception: start it with the schedule call and no `scheduled_at`, and end it with `/end`.

Pause and resume are on the API: `POST /v1/email/campaigns/{id}/pause` (from `scheduled`, `compiling`, or `sending`; no body, any field is `400`; sets `status: paused`, `pause_reason: api`) and `POST /v1/email/campaigns/{id}/resume` (optional body `{ "resume_at": "<ISO 8601 date-time>" | null }`). A pause with reason `api`, `manual`, or `provider_refused` waits for a resume call (for `provider_refused`, after support fixes the cause); the other reasons (`insufficient_balance`, `insufficient_funds_ancestor`, `sender_inactive`, `no_content`, `tenant_paused`, `gmail_spam_rate`) resolve on their own once the cause clears. A resume while the cause persists pauses the campaign again. Already paused on pause, or already running on resume, returns `200` with the current state. A `resume_at` more than a minute ahead keeps the campaign `paused` and resumes it at that time (the campaign's `resume_at` field shows it; null otherwise; a later pause cancels it); a past `resume_at` is `400`. Resume checks sending controls (`409 SENDING_PAUSED`) and returns `409 EMAIL_CAMPAIGN_TEST_REQUIRED` when content changed during the pause and was not tested; send the test from the dashboard, then resume.

**Email is generally available.** Write endpoints under `/v1/email/*` require the `email` entitlement on the key's organization and return `403 ENTITLEMENT_REQUIRED` with `details.entitlement: "email"` without it. Reads are open to any key. That response is not a bad key or a permissions problem: do not retry it and do not tell the operator their credentials are wrong; the organization needs email enabled on its account, so escalate to a human.

`POST /v1/email/campaigns/{id}/schedule` can additionally return `409 SENDING_PAUSED` when sending is paused. Escalate it to a human operator rather than retrying.

Facts that differ from the messaging surface, and that agents get wrong if they assume otherwise:

- **Email endpoints are keyset paginated.** They return `{ "data": [...], "has_more": bool, "next_cursor": string|null }` inside `data`, unlike the messaging endpoints, which return a plain array. Page until `next_cursor` is null. Cursors are opaque: never parse, construct, or reuse one across a different ordering.
- **There is no inbox and no inbound email.** We do not receive email, and there is no inbound email webhook. Replies go to the `reply_to` address on the sender identity. Do not build a polling loop looking for one.
- **There is no A/B testing.** The `email.opened` webhook event exists but is approximate (mail apps that hide opens are not counted) and opt-in: never treat it as a count of readers.
- **DNS is manual.** `GET /v1/email/domains/{id}` returns the records to publish; the platform never writes DNS and never asks for registrar credentials. Domains are registered in the dashboard. Poll the domain until `status` is `active`.
- **Sender identities carry a read-only `gmail_verified_sender` object.** `{ status, submitted_at, verified_at }`, where `status` is one of `not_eligible | eligible | ready_to_submit | submitted | verified | suspended | rejected | expired` and `verified_at` is set only when `status` is `verified`. It reflects the sender's standing in Google's Gmail Verified Sender Program, submitted through Campaign Verify; it is never a sending gate.
- **Use suppressions to stop mailing an address.** `POST /v1/email/suppressions` is the public primitive; it survives a re-import, because the underlying contact rows carry the bounce and complaint history. There is no public contact-delete. Scope is `org`, `identity`, or `domain`, and all three apply automatically - there is nothing to attach to a campaign. To hold a set of people out of ONE campaign instead, put any list with emails in the campaign's `suppression_list_ids`: any list can be an audience or a hold-out, and the campaign decides which.
- **Lists are one family for both channels.** `/v1/contact-lists` carries phones and emails; the `/v1/email/lists` endpoints are removed (404). Every list object has `channels` and an `email` object (status, counts, quality grade; null when the list has no emails). `GET /v1/contact-lists` defaults to `channel=text`, so ask for `channel=email` or `all` to find email lists. Emails reach a list only through `POST /v1/contact-lists/import` with an `email_column` (and `phone_column` for a file with both), which creates the list: there is no endpoint to create an empty list or to add or page through a list's emails. An import with an `email_column` needs the email entitlement of the list's organization. List ids are what `/v1/email/campaigns` takes in `list_ids` and `suppression_list_ids`; contact tags (`tag_ids`, `require_tag_ids`, `suppression_tag_ids`) work beside them, see Tags.
- **Check before scheduling.** `GET /v1/email/campaigns/{id}` returns a `blocked` array naming exactly what is stopping a schedule. Read it rather than calling schedule and interpreting the failure.
- **Importing a list is one call over a URL you host, and the file IS the list.** `POST /v1/contact-lists/import` is the one list import, for phone numbers, emails, or both. It fetches an HTTPS CSV (50 MB cap, SSRF-guarded), checks the columns (each must be a header of the file, and its first rows must hold phone numbers or email addresses, else `400` and nothing is written), and commits it as a NEW list, answering `202` before any row is written. Send `list_name` and `phone_column`, `email_column`, or both; the email column is never guessed. `consent` (`source`, optional `note`) is optional and allowed only for an audience with an `email_column`; omit it and none is stored. `email_domain_id` scopes its emails to one sending domain (omit it for a list any campaign may use), and `acquired: true` (optional, default false) marks a purchased or rented list, which is sent more gradually. With `merge_tags`, phones map only the named columns while emails also map every remaining column by its default name (email custom field names match `^[a-z][a-z0-9_]{0,63}$`). Unknown top-level body fields are ignored; unknown keys inside `merge_tags` or `consent` return `400`. `email_column` needs the email entitlement (`403 ENTITLEMENT_REQUIRED`). Poll `GET /v1/contact-lists/{id}` for both sides: `import.status` for phones and `email.status` for emails, each until `ready` or `failed`. When one is `failed`, read `import.error` (or `email.error`): `{ code, message }`, `null` unless failed, with `code` `TAG_COLUMN_TOO_MANY_TAGS` for a fixable tag column or `IMPORT_FAILED` otherwise; show the `message` to the user, and do not retry blindly. `purpose: "do_not_contact"` adds phones to an opt-out list and emails to unsubscribes (`GET /v1/email/suppressions`), and returns no `list_id`. A column that is not a header returns `400` with `details.headers`, so fix the column name rather than retrying the same body. Role addresses (info@, admin@) are always removed at import. `POST /v1/email/lists/import` was removed: send `name` as `list_name`, the `mapping` entry valued `email` as `email_column`, and the rest of `mapping` as `merge_tags`.
- **Every campaign email is screened at send time; there is no validation step.** An address rated as likely undeliverable is not sent to: its message status is `suppressed`, it is not counted as a bounce, no `email.bounced` webhook fires, and it is held out of sends for 90 days, then re-checked. Campaigns have no recipient policy: they send to every subscribed, unsuppressed contact. Lists and contacts carry no `validated_at` or `validation_state`. Yahoo-run mailboxes (Yahoo, AOL, AT&T, Verizon, Cox, Frontier) never join a list: imports remove them and count them in the list's `email.removed_at_upload` under `yahoo_family`. Recipients at providers whose mailboxes report spam at an unusually high rate (currently windstream.net) are removed the same way, counted under `high_complaint_domain`.
- **Read `lint` on every template write.** `POST /v1/email/templates` returns a `lint` object beside the template. A template with lint errors saves, but a campaign built on it will not schedule, so a caller that ignores `lint` discovers the problem at schedule time instead.

Nine email webhook events subscribe alongside the message events (listed under Webhooks below). Each payload carries a stable `id` that is also the deduplication key.

## Rate limits and backoff

- **Limit:** 600 requests per minute per API key, the same for every scope: bursts of up to 600 at once, refilling at 10 per second.
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
- `contact_list.analyzed` (opt-in; fires when a list finishes analysis; payload `id`, `list_id`, `name`, `analysis`, `analyzed_url`)

Email events:

- `email.delivered`
- `email.bounced` (permanent bounces only; soft bounces are deferred and never fire)
- `email.complained`
- `email.unsubscribed`
- `email.clicked` (unique, non-bot clicks)
- `email.replied` (a reply came back through the platform: an email of a campaign with follow-ups, or a campaign a triggered campaign listens to)
- `email.sent` (accepted for delivery)
- `email.failed` (`reason` is `rejected` or `render_failed`)
- `email.opened` (opt-in, not in the default set; the first open of a message; approximate, because mail apps that hide opens are not counted)

**Signature verification is mandatory.** Each delivery carries an HMAC-SHA256 signature in the `X-Webhook-Signature` header, formatted `sha256=...`. Compute the HMAC-SHA256 of the raw request body with your webhook secret, compare against the header using a constant-time comparison, and reject anything that does not match before touching the payload.

**Delivery semantics:** respond with a 2xx quickly. Non-2xx responses trigger automatic retries, so design handlers to be idempotent on event delivery. An endpoint that fails every delivery for 3 days is turned off and the organization's owners and admins are emailed; a human re-enables it in the dashboard after fixing the URL, and events from while it was off are not resent.

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
