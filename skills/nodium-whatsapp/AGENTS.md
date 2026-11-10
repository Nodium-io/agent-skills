# Nodium — Agent Skill

Nodium is an API gateway that connects a developer's backend to conversational
surfaces: the official WhatsApp Cloud API. Nodium never
answers for you: it carries messages, your backend decides the reply.
One account, one key, one client per end customer (plus your own company),
HTTPS only.

Full reference: `https://nodium.io/en/docs/api` · every route in one file:
`https://nodium.io/docs/api/llms.txt` · OpenAPI 3.1:
`https://nodium.io/docs/api/openapi.json` · everything in one file:
`https://nodium.io/llms-full.txt` · status: `https://nodium.io/status.json`.

## SDK

```
npm i @nodium.io/whatsapp     # Node 20+, ESM and CommonJS, typed
npx @nodium.io/cli listen     # forward webhooks to localhost while you build
npx @nodium.io/cli status     # what your key sees, and the next step
```

The `nodium` command (`@nodium.io/cli`) prints JSON when piped (`--output json`).
As an agent: read first (`status`, `numbers list`, `templates get <id>`), and
pass `--yes` to a command that sends or changes something only once the
person has approved it. Guide: `https://nodium.io/en/docs/api/guide/cli`.

```ts
import { Nodium } from '@nodium.io/whatsapp'
import { verifyWebhook } from '@nodium.io/whatsapp/server'

const nodium = new Nodium(process.env.NODIUM_KEY!)
await nodium.messages.send({ channelId, to: '+33612345678', type: 'text', text: 'Hello', idempotencyKey: 'order-4187' })
// One client among yours: nodium.as('<client id or externalId>').messages.send(…)
```

Errors are `NodiumError` (`code`, `status`, `requestId`, `retry`): branch on
`code`, never on the message. `verifyWebhook(rawBody, headers, secret)` checks the
signature and returns the event (it throws `NodiumSignatureError` otherwise); `receiveFlowRequest` / `respondToFlow` serve a dynamic Flow.

## MCP server

Operate WhatsApp from an agent through the Model Context Protocol, with the
same key: `https://nodium.io/api/v1/mcp` (Streamable HTTP, POST only, no
session).

```
claude mcp add --transport http nodium https://nodium.io/api/v1/mcp --header "Authorization: Bearer $NODIUM_KEY"
```

Each tool calls the matching route with your key (`send_message`,
`list_conversations`, `list_messages`, `list_templates`…); to confine an
agent, give it its own key pinned to one client, with an expiry. The only writes are `send_message`, `mark_read` and `update_contact`;
broadcasts, templates, clients and connect links stay with the API. `send_message` is served only when the URL ends with
`?send=true` (`https://nodium.io/api/v1/mcp?send=true`), so a message read by the agent cannot make it send. Every tool takes an optional `tenant` (client id or
`externalId`). Results are the content of `data` as JSON text; refusals come
back with `isError: true` and the API's `error.code`. Guide:
`https://nodium.io/en/docs/api/guide/mcp`.

## Base URL and envelope

```
https://nodium.io/api/v1
```

- Success: `{ "data": ... }`.
- Refusal: `{ "error": { "status": 409, "code": "window_closed", "message": "...", "detail": "...", "param": null, "doc_url": "...", "requestId": "..." } }`.
  Test `error.code` (stable), never the text. `param` names the refused field of
  the body or query (`null` when it is not about one field); `doc_url` points at
  the explanation of the code in the guide (`null` for a code it does not list).
  Quote `requestId` when asking for help.
- Every list answers `items`. A paginated list (`limit`, then send `nextCursor`
  back as `cursor` until it is `null`) also gives `nextCursor`, `hasMore` and,
  on its first page when it counts, `total`. A list returned whole (a bounded
  set) gives `items` only.
- Every response carries `X-RateLimit-Limit`, `X-RateLimit-Remaining`,
  `X-RateLimit-Reset`.

## Authentication

`Authorization: Bearer <key>`.

- **The account key** (`nod_…`): one kind of key, owned by your account and
  created in the console (Settings › API keys) or with `POST /keys`. It acts as
  itself (messages carry its label) and opens the whole published API of your
  account. A pinned key creates no other key, and a key created by a key never
  expires after it. The secret is shown once.
- **The client it acts on**: the client the key is pinned to (`clientId` at
  creation), otherwise the one named by `Nodium-Tenant: <client id or
  externalId>`, otherwise your own company ("My company"). A client outside
  your account answers `404`, never `403`. A key pinned to one client cannot
  name another.
- **No roles**: every key opens the same API. Routes that cover every client
  (`/tenants`, `/keys`, `GET /channels?scope=account`) need a key that is not
  pinned (`all_clients_required`).

## Resources

### Conversations and messages

- `GET    /conversations` — list (cursor; `channelId`, `contactId`, `phone`: the contact with this number, written any way, 6 digits at least)
- `GET    /conversations/:id` — one conversation, with its `window` (`open`, `expiresAt`, `minutesLeft`)
- `GET    /conversations/:id/messages` — history (cursor)
- `POST   /messages` — **the one way to send**, by channel and person (`channelId`, `to` or `recipient`, `type`, `idempotencyKey`); Nodium finds or opens the conversation (see below)
- `POST   /media` — upload a file (multipart: `file`); returns a `mediaId` valid 7 days, sent with `POST /messages`
- `POST   /conversations/:id/read` — read receipt
- `POST   /conversations/:id/typing` — typing indicator
- `GET    /messages/:id` — one message as it stands (status, `errorInfo`, ids)
- `GET    /messages/:id/media` — download a received file

### Contacts

- `GET    /contacts` — list (cursor, `q`, `consent`, `blocked`, `channel`)
- `GET    /contacts/export` — the same filters, as a CSV file (10,000 rows at most)
- `POST   /contacts` — create
- `GET    /contacts/:id`
- `PATCH  /contacts/:id` — fields, marketing consent
- `DELETE /privacy/contacts/:id` — erase a person, for good

### Templates

A template belongs to the client you act on and is deployed to the WhatsApp
accounts of its numbers, including numbers it connects later (Meta reviews it
in each). To send one, its name is enough: Nodium picks the approved version of
the client. One button can open a WhatsApp Flow (`{ "type": "FLOW", "text",
"flowId", "screen" }`): the only way to open a Flow outside the 24-hour window.

- `GET    /templates` · `GET /templates/:id` — `status` is for the client you act on; `statusCounts` and `clients` give the state per client
- `POST   /templates` — draft, for the client you act on; nothing reaches Meta before `deploy`
- `PATCH  /templates/:id` — edit while no client holds it at Meta · `DELETE /templates/:id`
- `POST   /templates/:id/deploy` — deposit it at every client it targets; only does what is left to do, safe to repeat
- `POST   /templates/:id/refresh` — ask Meta for the state at each client
- `POST   /templates/import` — import the templates already at Meta

### Broadcasts

- `POST   /broadcasts/preview` — who a broadcast would reach
- `POST   /broadcasts` — one template to many contacts; refused (`422 broadcast_exceeds_messaging_limit`) when it exceeds the number's messaging tier per 24 h
- `GET    /broadcasts` · `GET /broadcasts/:id` — `status`: `sending` · `paused` · `finished` · `cancelled`; a recipient is `sent`/`failed` only once its message is settled
- `POST   /broadcasts/:id/cancel`
- `POST   /broadcasts/:id/resume` — continue a broadcast the circuit breaker paused (sends refused over and over for an account reason: payment, quality, rate limit, token, plan allowance); `409 broadcast_not_paused` otherwise

### Channels and numbers

- `GET    /channels` — the channels of the client you act on (WhatsApp numbers); `?scope=account` for every client
- `POST   /channels` — create a test channel
- `GET    /channels/:id` — one number: quality, sending tier, last health check
- `PATCH  /channels/:id` — switch on or off, rename · `DELETE /channels/:id`
- `POST   /channels/:id/check` — check a WhatsApp number
- `POST   /channels/:id/diagnostic` — why a number does not receive or send: eight checks, each with what to do and who can fix it (6 per minute per number)
- `GET    /channels/:id/profile` · `PATCH /channels/:id/profile` — the business profile read at Meta: `about`, `description`, `address`, `email`, `websites[]`, `vertical`; only the fields you send change

Numbers are connected by the customer through Meta's Embedded Signup, from a
hosted connect link (see Clients). A channel with `paymentMissing: true` has a
WhatsApp account without a payment method at Meta (error 131042 on paid sends). Coexistence (the
WhatsApp Business app and the API on the same number) is supported.

### Webhooks

A subscription belongs to the client the call acts on. One created on "My
company" also receives the events of every client of your account; every
payload carries the originating client's id in `workspace_id`.

- `GET    /webhooks` · `POST /webhooks` (`url`, `events[]` or `["*"]`, `label`, `channelIds[]`) → the secret, shown once
- `PATCH  /webhooks/:id` — url, events, label, `channelIds`, `isActive` (pause/resume) · `DELETE /webhooks/:id`

`channelIds` limits a subscription to some numbers (empty: all). An event that
belongs to no number (a template status, a consent change) is not delivered to
a limited subscription.
- `POST   /webhooks/:id/test` — one signed test call now
- `POST   /webhooks/:id/rotate-secret` — new secret, old one still signs for 24h
- `GET    /webhooks/deliveries` — delivery log
- `POST   /webhooks/deliveries/:id/redeliver` — same payload, same event `id`

### Clients

A client is one of your customers: its own numbers, contacts, conversations and
templates, isolated from the others. "My company" is your own, born with the
account (not listed).

- `GET    /tenants` · `POST /tenants` (`name`, `externalId`, `locale`, `timezone`) → `{ tenant }`
- `GET    /tenants/:id` · `PATCH /tenants/:id` · `DELETE /tenants/:id` — `:id` is the client id or your `externalId`
- `POST   /tenants/:id/connect-links` — hosted connect link (`redirectUrl`, `expiresInDays`)
- `GET    /tenants/:id/channels`
- `GET    /connect-links` — the hosted links of the account, never their address (`state`, `client`, cursor) · `DELETE /connect-links/:id` — an unused link stops working

### Keys

- `GET    /keys` · `POST /keys` (`label`, `clientId` to pin it, `expiresInDays` 1–3650, absent = never expires) → `{ key, token }`, the token shown once; past its date a key answers 401 `api_key_expired`
- `GET    /keys/current` — the key making the call: `{ id, accountId, pinned, expiresAt }` (any key)
- `DELETE /keys/:id` — revoke

### WhatsApp Flows

A Flow is a form shown inside WhatsApp; it lives at Meta and Nodium keeps its
versions. A draft can be edited, a published Flow only deprecated or
duplicated. Answers arrive as `message.received` (form reply), or through
`GET /flow-responses`.

- `GET    /flows` · `POST /flows` (`name`, `categories[]`, `flowJson`, `channelId` — the number whose WhatsApp account hosts the Flow) · `GET /flows/:id` · `PATCH /flows/:id` · `DELETE /flows/:id` (draft only)
- `POST   /flows/sync` — read the client's Flows back from Meta
- `GET    /flows/:id/versions` · `POST /flows/:id/versions` (`flowJson`, validated by Meta: read `validationErrors`) · `GET /flows/:id/versions/:versionId`
- `GET    /flows/:id/preview` — Meta's preview link
- `POST   /flows/:id/publish` — final · `POST /flows/:id/deprecate` · `POST /flows/:id/duplicate`
- `GET    /flows/:id/data-endpoint` · `PUT` (`url`) → the signing `secret`, shown once · `DELETE` — a dynamic Flow: Meta calls Nodium, which relays to your `url`
- `POST   /flows/:id/data-endpoint-test` — sends your `url` a signed `INIT` call (`test: true`) and returns status, duration and your answer
- `GET    /flows/encryption` · `POST /flows/encryption` (`channelId`, optional) — whether the client's encryption key is registered at Meta, per number, and registering it
- `GET    /flows/:id/invocations` — the last calls Meta made (no content)
- `GET    /flow-responses` — what contacts entered (`flowId` for one Flow, `flowToken`, `from`, `to`, cursor); `GET /flows` gives each Flow's `responses` count

### Files and statistics

- `GET    /media` — the files Nodium keeps (`kind`, `from`, `to`, cursor) · `DELETE /media/:id` — remove the copy (the message stays)
- `GET    /stats/delivery` — sent, delivered, read, replied, failed, skipped per day and per template or broadcast (`from`, `to`, `source`, `channelId`; 366 days at most)

### Event log

The webhook is a doorbell, the event log is the truth: every event Nodium emits is also written to a numbered log, kept 30 days.

- `GET    /events` — `after` (a `seq`), `limit` (≤ 500, default 100) → `{ data: [envelope + seq], next_after }`. Without `after`, `data` is empty and `next_after` is the head: store it as your first cursor, then loop while `next_after` moves. Entries come in order, none skipped; deduplicate on `id` (the same as in webhook deliveries). A key that sees every client reads them all (or the one `Nodium-Tenant` names); a pinned key, one client. A cursor older than the log answers `410 cursor_expired`.

### Other

- `GET    /billing/summary` — this month: active clients, messages and tool calls against the allowance, estimated total
- `GET    /api-requests` — your recent calls (30 days, no bodies)
- `GET    /health` — is everything working
- `GET    https://nodium.io/status.json` — public status (outside `/api/v1`)

## Sending a message

One call sends everything, by **channel and person** — never a conversation id:

```
POST /api/v1/messages

channelId       string   REQUIRED  which number speaks (GET /channels)
to | recipient  string   REQUIRED  the person: a phone number in E.164 (`to`), or their
                                   WhatsApp user id `wa:…` (`recipient`) — one of the two
idempotencyKey  string   REQUIRED  your unique key, 8–200 chars (or the `Idempotency-Key` header,
                                   same value if both are given)
type            string   REQUIRED  text | interactive | location | contacts | template |
                                   image | video | audio | document | reaction
replyToId       string   optional  quote an earlier message of this conversation (Nodium id)
queue           boolean  optional  return 202 "queued" and send within a minute
scheduledAt     string   optional  ISO 8601, at least 1 minute ahead → 202 "scheduled"
```

Each `type` takes its own fields (a field that does not belong is refused):

```
text         text                              ≤ 4,096 characters
interactive  form (+ text)                     buttons (1–3), list (≤ 10 rows), location_request, link or flow
location     location                          latitude, longitude, name?, address?
contacts     contacts                          one contact card: name.formatted_name required
template     templateName (+ templateLanguage) | templateId, variables, buttons
image…       mediaId, caption?, voice?         the id from POST /media; caption ≤ 1,024 (none for audio)
reaction     messageId, emoji                  emoji "" removes; no message is created
```

Nodium finds the conversation for (channel, person), or opens it — a person it has
never seen becomes a contact. Returns `{ conversationId, message, duplicate }`
(`message` is `null` for a reaction). Same `idempotencyKey` twice with the same
request = the first message returned with `duplicate: true`, never a second send; the same
key with different content (number, recipient, type, text, template values…) =
`409 idempotency_key_conflict`, nothing sent (`queue` and `scheduledAt` are not compared).

**A refused send is still HTTP 200/201**: test `message.status` (`failed`); `message.error`
is the provider's reason and `message.errorInfo` = `{ code, title, action, actor }` explains it
(`actor`: `you` | `client` | `nodium` | `wait`; `null` when it did not fail). `GET /messages/:id`
re-reads a message later.

**The 24h window**: free-form types (everything but `template` and `reaction`) are
refused with `window_closed` (409) more than 24 hours after the person last wrote —
and to anyone who never did. Send `type: "template"` with `templateName` (or
`templateId`), `variables` (`{ "1": "Karim" }` or `{ "client_name": "Karim" }`), and
optionally `buttons`.

**Files**: `POST /media` first (multipart `file`, 4 MB at most) → `mediaId`, valid for
7 days and sendable to many people; then `POST /messages` with `type` set to the
returned `kind`.

Transient refusals from Meta (429, 5xx) are retried by Nodium after 30 s,
2 min and 10 min before `message.failed`.

## Webhooks

### Events

`message.received`, `message.imported`, `message.echoed`, `message.edited`,
`message.revoked`, `message.sent`, `message.delivered`,
`message.read`, `message.failed`, `conversation.opened`, `contact.opted_in`,
`contact.opted_out`, `template.status_changed`, `channel.connected`,
`channel.connect_failed`, `channel.health_changed`, `history.progress`. Subscribe to `*` for all.
`message.imported` (the history of a coexistence number) is written to the event
log only: it never rings.

Payload: `{ "id": "evt_…", "event": "...", "workspace_id": "...", "external_id": "...", "occurred_at": "...", "data": { ... } }`
(`external_id`: your id for the client, `null` for "My company").
`message.received` carries `message_id`, `conversation_id`, `contact_id`,
`channel_id`, `provider_message_id`, `from`, `profile_name`, `type`, `text`,
`reply` (button/list choice), `location`, `media` (`url`, `mime_type`,
`filename`), `window_expires_at` (end of the 24 h window), `blocked` (a blocked contact: stored and
announced, nothing should answer), and on a reaction `reaction_to` and `emoji`;
a change of number arrives as `type: "system"` (`previous_number`,
`new_number`). `message.sent` and `message.failed` carry the whole content, the
`idempotency_key` and `to`, whoever sent it; `message.failed` adds `error_code` and
`error_info` (`{ code, title, action, actor }`). `channel.connected` carries
`phone_number_id`.

### Signature

Headers: `X-Nodium-Event`, `X-Nodium-Delivery`,
`X-Nodium-Signature: t=<unix-seconds>,v1=<hex>`.

Verify `HMAC-SHA256(secret, "<t>.<raw body>")` on the raw body, before JSON
parsing. During a rotation two `v1=` values appear (newest first) — accept if
any matches. Reject timestamps older than 5 minutes. Deduplicate on the
payload `id` (stable across retries and redeliveries).

### Delivery

- Answer `2xx` within 10 seconds.
- 6 attempts: after 1 min, 5 min, 30 min, 2 h, 12 h; then `failed`.
- Ordered per conversation (an event waits for older events of the same conversation).
- `isActive: false` pauses (deliveries wait). Nodium pauses a subscription on
  its own after 50 consecutive failures with nothing delivered for 24 h
  (`disabledReason`); setting `isActive: true` resumes it.

## Clients (multi-tenant)

1. `POST /tenants` with your `externalId` (replay → `409 tenant_exists` with the existing id).
2. `POST /tenants/:id/connect-links` → `url` of a hosted page (`/connect/<token>`),
   single use, 7 days by default (30 max). Your customer connects their number
   without a Nodium account, is guided to add a payment method at Meta
   (recommended, skippable), and lands on
   `redirectUrl?status=connected&channel_id=…` (or `status=cancelled`). The
   `channel.connected` webhook fires too.
3. Act inside the client with `Nodium-Tenant: <id or externalId>` and your key,
   or with a key pinned to it (`POST /keys` with `clientId`).

The plan follows connected WhatsApp numbers and volume, for the whole account
(clients are free). Build (free): 3,000 messages a month and 1 number; at the
allowance it blocks sends with `402 quota_exceeded` (reception continues) and
refuses one more number with `402 plan_limit`. Launch and Scale include more
messages and numbers, then bill extra numbers and blocks of 1,000 messages, up
to the overage cap the account sets. Meta bills its fees to each customer's
WhatsApp Business account, with no markup from Nodium.

## Key constraints

- **Idempotency key** required on every send.
- **Text limits**: 4,096 characters for a text, 1,024 for a file caption — refused at once (`text_too_long`, `caption_too_long`, 422), nothing stored.
- **24h window** enforced on free text (`window_closed`, 409). Only a message
  from the customer reopens it; a template never does.
- **Rate limits**: 600 calls/min per client (per key when the key is pinned), 6,000/min per key (`429 rate_limited`, `Retry-After`);
  30 free-text replies (text and files)/min per key and per client;
  templates are not limited by it.
- **Marketing opt-out**: a contact who opted out (or Meta error 131050) gets
  `consent_missing` on marketing templates.
- **HTTPS only** for webhook and app URLs.

## Error codes (selection)

The full list is in the reference (`/en/docs/api/guide/responses`).

- `window_closed` — 24h window closed, send a template
- `idempotency_key_required` · `idempotency_key_conflict` (same key, different content)
- `no_recipient` · `not_whatsapp` · `contact_blocked` · `consent_missing`
- `template_required` · `template_not_approved` · `template_variables_invalid` · `template_holes_unfilled` · `template_not_applicable` · `template_locked` · `template_category_locked` · `template_edit_refused` · `template_exists`
- `file_missing` · `file_too_large` · `file_type_refused` · `caption_unsupported`
- `channel_required` · `unknown_channel` · `no_channel`
- `webhook_url_not_https`
- `plan_limit` · `quota_exceeded` · `rate_limited`
- `tenant_required` · `tenant_invalid` · `tenant_not_found` · `tenant_exists` · `all_clients_required`
- `link_not_found` · `link_used` · `link_expired` · `link_closed`
