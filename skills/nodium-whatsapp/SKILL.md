---
name: nodium-whatsapp
description: Integrate WhatsApp into a project with Nodium, the WhatsApp API gateway (Meta's official Cloud API). Use when the task is to send or receive WhatsApp messages, verify Nodium webhooks, work with templates, Flows or broadcasts, or connect a SaaS's customers' WhatsApp numbers.
---

# Nodium WhatsApp

Read `AGENTS.md` in this folder before writing any code: it is the integration sheet for Nodium (keys, sending, the 24-hour window, webhooks and their signature, clients, error codes).

Rules that hold for every integration:

1. Read the API key from `NODIUM_KEY`, on the server only. Never in a browser or a mobile app.
2. Verify the webhook signature on the raw body, before any JSON parsing (`verifyWebhook` from `@nodium.io/whatsapp/server`, or an HMAC-SHA256 check in any other language).
3. Every send carries an `idempotencyKey`.
4. Free text only goes out within 24 hours of the customer's last message; outside it, send an approved template.
5. Branch on `error.code`, never on the error message. A send refused by WhatsApp is still HTTP 200: test `message.status`.

More, when you need it:

- Every route in one file: https://nodium.io/docs/api/llms.txt
- The whole API guide: https://nodium.io/llms-full.txt
- OpenAPI 3.1: https://nodium.io/docs/api/openapi.json
