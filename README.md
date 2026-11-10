# Nodium for AI coding agents

Everything a coding agent needs to integrate WhatsApp with [Nodium](https://nodium.io), the WhatsApp API gateway for developers: the integration sheet, a Claude Code skill, the MCP server configuration and a ready-to-paste prompt.

| File | What it is |
|---|---|
| [`AGENTS.md`](./AGENTS.md) | The integration sheet: keys, sending, the 24-hour window, webhooks, clients, errors. Drop it at the root of your repository. |
| [`skills/nodium-whatsapp/`](./skills/nodium-whatsapp) | The same, packaged as an Agent Skill for Claude Code. |
| [`mcp.json`](./mcp.json) | The MCP server block, for Cursor and any MCP client. |
| [`PROMPT.md`](./PROMPT.md) | A prompt that makes an agent integrate WhatsApp into your project. |

## Give your agent the integration sheet

```bash
curl -o AGENTS.md https://nodium.io/agents.md
```

Codex, Cursor and Claude Code read `AGENTS.md` at the root of a repository.

## Install the Claude Code skill

```bash
git clone https://github.com/Nodium-io/agent-skills
mkdir -p .claude/skills
cp -r agent-skills/skills/nodium-whatsapp .claude/skills/
```

## Connect the MCP server

Nodium serves an MCP server at `https://nodium.io/api/v1/mcp`: your agent reads numbers, conversations, contacts, templates and the journal with your API key.

```bash
# Claude Code — reads only
claude mcp add --transport http nodium https://nodium.io/api/v1/mcp \
  --header "Authorization: Bearer $NODIUM_KEY"

# Letting the agent send messages too (send_message is off without ?send=true)
claude mcp add --transport http nodium-send "https://nodium.io/api/v1/mcp?send=true" \
  --header "Authorization: Bearer $NODIUM_KEY"
```

For Cursor, copy [`mcp.json`](./mcp.json) to `.cursor/mcp.json` and put your key in it.

- Sending is off by default: an agent that only reads cannot be pushed into writing to anyone by a message it received.
- Broadcasts, templates and clients are not exposed as tools: they stay with the API, called by your own code.
- To bound an agent to one of your customers, give it a key pinned to that client.

Guide: https://nodium.io/en/docs/api/guide/mcp

## Text for LLMs

- https://nodium.io/llms.txt — the site, summarised
- https://nodium.io/llms-full.txt — the summary, `AGENTS.md` and the whole API guide
- https://nodium.io/docs/api/llms.txt — every published route
- Every guide of the site also reads as Markdown: add `.md` to its address.

## Related

- SDK and examples: https://github.com/Nodium-io/sdk
- OpenAPI description: https://github.com/Nodium-io/openapi

MIT © By-Tech.io
