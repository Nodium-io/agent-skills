# The integration prompt

Paste this into your coding agent (Claude Code, Cursor, Codex) at the root of your project.

```text
Integrate WhatsApp into this project with Nodium.

1. Before any code, read:
   https://nodium.io/agents.md
   The whole API guide: https://nodium.io/llms-full.txt
2. Install the SDK:
   npm i @nodium.io/whatsapp
   Another language: the REST API, described in
   https://nodium.io/docs/api/openapi.json
3. Read the key from NODIUM_KEY.
   Never in a browser or an app.
4. Receive: one POST route that checks the signature
   on the raw body (verifyWebhook, from
   @nodium.io/whatsapp/server), then handles
   message.received.
5. Send: messages.send, an idempotencyKey each time.
   After 24 hours of silence, an approved template.
6. Branch on error.code, never on the message.

Write the tests, and list the environment variables.
```
